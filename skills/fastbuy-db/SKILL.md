---
name: fastbuy-db
description: Work with the BSC meme-token swap database (Four.meme and Flap) through the fastbuy-db MCP server. Use it when asked about tokens, swaps, wallets, volumes, pools or indexer state; when writing SQL against this database; or when the tables four_tokens, flap_tokens, four_curve_swaps, flap_curve_swaps, four_dex_swaps, flap_dex_swaps, dex_pools, indexer_state come up.
---

# The fastbuy swap database

An index of BSC meme-token swaps. Four swap streams, each in its own table:

|                | Four.meme          | Flap               |
| -------------- | ------------------ | ------------------ |
| bonding curve  | `four_curve_swaps` | `flap_curve_swaps` |
| DEX afterwards | `four_dex_swaps`   | `flap_dex_swaps`   |

The indexer writes them; everyone else reads. Through this MCP server the access is
read-only: any write is rejected by the database role itself.

## Three things people trip over

**1. Addresses and hashes are `BYTEA`, not text.**

```sql
-- to search: the index will be used
WHERE token = addr('0xbb4cdb…')
-- to display
SELECT hex(token) AS token
```

Or read the `<table>_hex` views — the same columns with addresses as strings:
`four_tokens_hex`, `flap_curve_swaps_hex`, and so on. `WHERE token = '0x…'` without
`addr()` will not work: the types do not match.

**2. Amounts are `NUMERIC(78,0)` in wei and arrive as strings.**

That is a `uint256`: 78 digits, which do not fit into a double. In code wrap them in
`BigInt`, never in `Number`. Dividing by `10^decimals` must use the precision of the
specific quote: every known one has 18, `XAUt` has 6. Never assume eighteen for a quote
you do not recognise.

**3. A wallet is matched against three fields at once.**

`maker` is who received the tokens, `tx_from` is who signed the transaction, `tx_to` is
who it was addressed to. A purchase made through the trader's own contract never lands in
`maker`, because the contract receives the tokens. So:

```sql
WHERE maker = addr($1) OR tx_from = addr($1) OR tx_to = addr($1)
```

All three fields are indexed. The `get_txs_by_maker` tool does this for you.

## The server's tools

Start with a ready-made query and reach for `execute_sql` when it is not enough.

| Tool               | When                                                              |
| ------------------ | ----------------------------------------------------------------- |
| `get_swaps`        | Swaps under any filter: token, wallet, side, quote, pool, period  |
| `get_txs_by_maker` | Every swap of an address — `maker`, `tx_from` and `tx_to` at once |
| `get_tokens`       | Find a token by symbol, name, address, creator or status          |
| `get_token`        | Token card: fields, pools, trade counters, volumes per quote      |
| `get_wallet`       | Wallet summary: trades, volumes, favourite tokens                 |
| `top_traders`      | Who traded the most over a period                                 |
| `list_tables`      | What the database holds and how much it weighs                    |
| `describe_table`   | Columns, types, indexes and foreign keys of one table             |
| `execute_sql`      | An arbitrary `SELECT` when no ready-made tool fits                |
| `indexer_status`   | How far the indexer got and whether it fell behind                |

## Do not

- **Treat an empty answer as an error.** Tokens launched before the indexer started are not
  in the database, and neither are their swaps. Before blaming the query, check
  `indexer_status`: the period you asked about may simply not be indexed.
- **Add up volumes across different quotes.** BNB, USDT and a tokenized stock are different
  units; they can only be summed one quote at a time.
- **Turn amounts into `Number`.** The precision is lost silently.
- **Search by `tx_hash` expecting an index.** The hash indexes were dropped (migration
  `0004`): such a search is a full table scan. Bound it by a period or a block range.
- **Trust `reltuples` as an exact count.** `list_tables` and `indexer_status` report the
  planner's estimate; for an exact number use `COUNT(*)` or the `count` parameter of
  `get_swaps`.

## Further reading

- [references/schema.md](references/schema.md) — every table, column, type and index, what
  each field means and what its `NULL` means.
- [references/recipes.md](references/recipes.md) — ready-made queries: wallet swaps, token
  volumes, fresh launches, indexer health.
