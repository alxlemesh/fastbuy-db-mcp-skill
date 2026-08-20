# Database structure

The schema is created and versioned by the `fastbuy-db` indexer; this describes it as
migrations `0001`–`0006` left it. The live answer always comes from `describe_table` — if
it disagrees with this text, it is right.

## General rules

- **Addresses and hashes are `BYTEA`.** An address is 20 bytes, a transaction hash 32.
  Compare with `= addr('0x…')`, display with `hex(col)`, or read the `<table>_hex` view.
- **Amounts are `NUMERIC(78,0)` in wei**, a `uint256` without loss. The driver returns them
  as strings.
- **Times are `TIMESTAMPTZ`.** `block_time` is the block's time, `indexed_at` is when the
  indexer wrote the row.
- **Enums:** `swap_side` is `buy` | `sell`; `dex_version` is `v2` | `v3`.
- **Functions:** `hex(BYTEA) → TEXT` (returns `0x…`), `addr(TEXT) → BYTEA` (the `0x` prefix
  is optional, case does not matter). Both are `IMMUTABLE`, so an index on the column is
  still used.

## Tokens

### `four_tokens` — Four.meme tokens

| Column                                      | Type            | What it is                                                         |
| ------------------------------------------- | --------------- | ------------------------------------------------------------------ |
| `address`                                   | `BYTEA` PK      | Token address                                                      |
| `creator`                                   | `BYTEA`         | Who launched it                                                    |
| `name`, `symbol`                            | `TEXT`          | From the `TokenCreate` event                                       |
| `decimals`                                  | `SMALLINT`      | Usually 18                                                         |
| `total_supply`                              | `NUMERIC(78,0)` | Total supply                                                       |
| `quote`                                     | `BYTEA`         | The curve's currency; read from the chain, not from events         |
| `request_id`, `launch_fee`                  | `NUMERIC(78,0)` | Launch bookkeeping                                                 |
| `launch_time`                               | `TIMESTAMPTZ`   | `TokenCreate.launchTime` — when trading opens                      |
| `tax_buy`, `tax_sell`                       | `NUMERIC(6,5)`  | Taxes as a fraction: `0.01` means 1%                               |
| `anti_sniper_fee`                           | `NUMERIC(78,0)` | `feeSetting`: `planId<<160 \| planAddress`                         |
| `max_offers`, `max_raising`                 | `NUMERIC(78,0)` | Curve ceilings                                                     |
| `enriched`                                  | `BOOLEAN`       | Whether chain fields were read; `false` means `quote` may be empty |
| `status`                                    | `TEXT`          | `curve` → `stopped` → `graduated`                                  |
| `trade_stop_block`, `graduated_block`       | `BIGINT`        | Where trading stopped and where graduation happened                |
| `graduated_at`                              | `TIMESTAMPTZ`   | When the token moved to a DEX                                      |
| `dex_pool`                                  | `BYTEA`         | The pool after graduation; `NULL` means still on the curve         |
| `created_block`, `created_tx`, `created_at` | —               | Where and when the token was created                               |
| `indexed_at`                                | `TIMESTAMPTZ`   | When the indexer wrote the row                                     |

Indexes: `created_at DESC`, `creator`, `lower(symbol)`, and a partial one on
`created_block` for tokens still awaiting enrichment.

### `flap_tokens` — Flap tokens

The same `address`, `creator`, `name`, `symbol`, `decimals`, `total_supply`, `quote`,
`tax_buy`, `tax_sell`, `dex_pool`, `created_*`, `indexed_at`, plus its own:

| Column                           | Type            | What it is                                                                                                                    |
| -------------------------------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `nonce`                          | `NUMERIC(78,0)` | The creator's launch number                                                                                                   |
| `meta`                           | `TEXT`          | Launch metadata                                                                                                               |
| `token_version`                  | `SMALLINT`      | Token contract version                                                                                                        |
| `migrator_type`                  | `TEXT`          | `V3_MIGRATOR`, `V2_MIGRATOR`, … — **never derive the pool version from it**: the migrator falls back from V3 to V2 on failure |
| `dex_id`, `lp_fee_profile`       | `SMALLINT`      | Where to migrate and with which fee profile                                                                                   |
| `curve_r`, `curve_h`, `curve_k`  | `NUMERIC(78,0)` | Curve parameters                                                                                                              |
| `dex_supply_thresh`              | `NUMERIC(78,0)` | The threshold for moving to a DEX                                                                                             |
| `max_buy_per_origin`             | `NUMERIC(78,0)` | Per-address purchase ceiling                                                                                                  |
| `progress`, `circulating_supply` | `NUMERIC(78,0)` | Curve progress and circulating supply                                                                                         |
| `status`                         | `TEXT`          | `curve` → `dex`, or `killed`                                                                                                  |
| `dex_pool_fee`                   | `INTEGER`       | Pool fee in hundredths of a percent: `2500` means 0.25%                                                                       |
| `launched_block`, `launched_at`  | —               | When the token reached a DEX                                                                                                  |

Indexes: `created_at DESC`, `creator`, `lower(symbol)`.

## Swaps

Four tables, all with the primary key `(block_number, log_index)` — the pair is unique
within the chain and is what makes a repeated sweep idempotent.

### Columns shared by all four

| Column                             | Type            | What it is                                                               |
| ---------------------------------- | --------------- | ------------------------------------------------------------------------ |
| `token`                            | `BYTEA`         | References `*_tokens(address)`, `ON DELETE CASCADE`                      |
| `side`                             | `swap_side`     | `buy` — the token was bought, `sell` — sold                              |
| `maker`                            | `BYTEA`         | Who received the tokens; a contract when trading through one             |
| `amount_token`                     | `NUMERIC(78,0)` | Token amount, in the token's `decimals`                                  |
| `amount_quote`                     | `NUMERIC(78,0)` | Quote amount, in the **quote token's** `decimals`                        |
| `quote`                            | `BYTEA`         | What the trade was denominated in                                        |
| `block_number`, `block_time`       | —               | The block and its time                                                   |
| `tx_hash`, `tx_index`, `log_index` | —               | Event coordinates; **there is no index on `tx_hash`** (migration `0004`) |
| `tx_from`, `tx_to`                 | `BYTEA`         | Who signed the transaction and who it was addressed to                   |
| `tx_value`, `gas_price`            | `NUMERIC(78,0)` | Transaction value and gas price                                          |
| `gas_limit`                        | `BIGINT`        | Gas limit                                                                |

Each table is indexed on `(token, block_time DESC)`, `(maker, block_time DESC)`,
`(block_time DESC)`, `(tx_from, block_time DESC)`, `(tx_to, block_time DESC)`.
**There is no index on `block_number`** — order and bound periods by `block_time`.

### What is specific to each

- **`four_curve_swaps`**: `price` — the price after the trade, `fee` — the fee, `offers`
  and `funds` — the curve's state after the trade.
- **`flap_curve_swaps`**: `fee`, `post_price` — the price after the trade, `portal_ts` —
  the time according to the Flap portal.
- **`four_dex_swaps`** and **`flap_dex_swaps`**: `pool` — the pool address, `version`
  (`v2`/`v3`), `fee_tier`, `amount0`/`amount1` — signed pool deltas (plus means the pool
  received), `sqrt_price_x96`, `liquidity`, `tick` (V3 only), `sender` — who called the
  swap (usually a router).

## Bookkeeping tables

### `dex_pools` — the pool registry

`address` PK, `protocol` (`four`/`flap`), `token`, `quote`, `version`, `fee_tier`,
`token_is_0` — whether the token is token0 in the pair (which decides the sign of
`amount0`/`amount1`), `source` — how the pool was discovered (`liquidity_added`,
`launched_to_dex`, `pair_created`, `pool_created`), `created_block`, `created_at`,
`active`.

### `indexer_state` — the cursor

A single row with `id = 1`: `last_block` — how far it got, `updated_at` — when it last
moved, and the all-time counters `sweeps`, `logs_read`, `swaps_written`, `swaps_skipped`.
`swaps_skipped` grows on swaps of tokens that are not in the database — that is normal,
not data loss.

## The `<table>_hex` views

`four_tokens_hex`, `flap_tokens_hex`, `dex_pools_hex`, `four_curve_swaps_hex`,
`flap_curve_swaps_hex`, `four_dex_swaps_hex`, `flap_dex_swaps_hex` — the same columns with
`BYTEA` shown as `0x` strings. They are the comfortable way to read; filtering is better
done on the tables themselves through `addr()`, because a view hides the index behind a
function.

## Quote precision

`amount_quote` is denominated in the **quote token's** `decimals`, not the traded token's.
Every known quote (WBNB, USDT, USD1, CAKE, BUSD, SOL, BTCB, U, UUSD and the Flap RWA
tokens) has 18; `XAUt` (`0x21caef8a43163eea865baee23b9c2e327696a3bf`) has **6**. Flap adds
new RWA quotes regularly, so an unfamiliar quote address is a reason to say "precision
unknown" rather than to divide by 10^18.
