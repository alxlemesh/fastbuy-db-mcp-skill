# Ready-made queries

Tested starting points for `execute_sql`. Addresses go in as parameters: that is what
lets an index be used, and it saves thinking about quoting.

Before writing SQL by hand, check whether a ready-made tool already covers it: `get_swaps`,
`get_txs_by_maker`, `get_token`, `get_wallet` and `top_traders` handle almost everything
below and deal with `BYTEA` and `decimals` for you.

## Every swap of a wallet over a period

All three fields at once: a purchase made through one's own contract never lands in
`maker`.

```sql
WITH wallet AS (SELECT addr($1) AS a)
SELECT 'four_curve' AS source, hex(s.token) AS token, t.symbol, s.side,
       s.amount_token, s.amount_quote, hex(s.quote) AS quote,
       s.block_time, hex(s.tx_hash) AS tx_hash, hex(s.maker) AS maker,
       hex(s.tx_from) AS tx_from, hex(s.tx_to) AS tx_to
  FROM four_curve_swaps s
  JOIN four_tokens t ON t.address = s.token, wallet w
 WHERE (s.maker = w.a OR s.tx_from = w.a OR s.tx_to = w.a)
   AND s.block_time >= now() - interval '24 hours'
UNION ALL
SELECT 'four_dex', hex(s.token), t.symbol, s.side, s.amount_token, s.amount_quote,
       hex(s.quote), s.block_time, hex(s.tx_hash), hex(s.maker), hex(s.tx_from), hex(s.tx_to)
  FROM four_dex_swaps s JOIN four_tokens t ON t.address = s.token, wallet w
 WHERE (s.maker = w.a OR s.tx_from = w.a OR s.tx_to = w.a)
   AND s.block_time >= now() - interval '24 hours'
UNION ALL
SELECT 'flap_curve', hex(s.token), t.symbol, s.side, s.amount_token, s.amount_quote,
       hex(s.quote), s.block_time, hex(s.tx_hash), hex(s.maker), hex(s.tx_from), hex(s.tx_to)
  FROM flap_curve_swaps s JOIN flap_tokens t ON t.address = s.token, wallet w
 WHERE (s.maker = w.a OR s.tx_from = w.a OR s.tx_to = w.a)
   AND s.block_time >= now() - interval '24 hours'
UNION ALL
SELECT 'flap_dex', hex(s.token), t.symbol, s.side, s.amount_token, s.amount_quote,
       hex(s.quote), s.block_time, hex(s.tx_hash), hex(s.maker), hex(s.tx_from), hex(s.tx_to)
  FROM flap_dex_swaps s JOIN flap_tokens t ON t.address = s.token, wallet w
 WHERE (s.maker = w.a OR s.tx_from = w.a OR s.tx_to = w.a)
   AND s.block_time >= now() - interval '24 hours'
 ORDER BY block_time DESC
 LIMIT 100;
```

## Trading through one's own contract

Rows where the tokens went to someone other than the transaction signer.

```sql
SELECT hex(maker) AS maker, hex(tx_from) AS signer, hex(tx_to) AS called,
       COUNT(*) AS trades
  FROM flap_curve_swaps
 WHERE maker <> tx_from AND block_time >= now() - interval '7 days'
 GROUP BY 1, 2, 3
 ORDER BY trades DESC
 LIMIT 20;
```

## Token volumes per quote

Different quotes cannot be added together — hence the grouping.

```sql
SELECT hex(quote) AS quote, side, COUNT(*) AS trades,
       SUM(amount_quote) AS volume_quote, SUM(amount_token) AS volume_token
  FROM flap_curve_swaps
 WHERE token = addr($1)
 GROUP BY 1, 2
 ORDER BY trades DESC;
```

## Launches of the last 24 hours

```sql
SELECT 'four' AS protocol, hex(address) AS address, symbol, name, status, created_at
  FROM four_tokens WHERE created_at >= now() - interval '24 hours'
UNION ALL
SELECT 'flap', hex(address), symbol, name, status, created_at
  FROM flap_tokens WHERE created_at >= now() - interval '24 hours'
 ORDER BY created_at DESC
 LIMIT 50;
```

## Tokens that made it to a DEX

```sql
SELECT hex(t.address) AS token, t.symbol, t.status, hex(t.dex_pool) AS pool,
       p.version, p.fee_tier, t.graduated_at
  FROM four_tokens t
  LEFT JOIN dex_pools p ON p.address = t.dex_pool
 WHERE t.status = 'graduated'
 ORDER BY t.graduated_at DESC NULLS LAST
 LIMIT 50;
```

Take the pool version from `dex_pools.version`, never from `flap_tokens.migrator_type`:
the migrator falls back from V3 to V2 on failure and then lies.

## Largest trades of a period in one quote

```sql
SELECT hex(s.token) AS token, t.symbol, s.side, s.amount_quote, s.block_time,
       hex(s.maker) AS maker
  FROM flap_curve_swaps s
  JOIN flap_tokens t ON t.address = s.token
 WHERE s.quote = addr($1)
   AND s.block_time >= now() - interval '24 hours'
 ORDER BY s.amount_quote DESC
 LIMIT 20;
```

## Activity by hour

```sql
SELECT date_trunc('hour', block_time) AS hour,
       COUNT(*) AS trades,
       COUNT(DISTINCT token) AS tokens,
       COUNT(DISTINCT maker) AS makers
  FROM flap_curve_swaps
 WHERE block_time >= now() - interval '24 hours'
 GROUP BY 1
 ORDER BY 1 DESC;
```

## The first trade of each token — who got there first

```sql
SELECT DISTINCT ON (s.token)
       hex(s.token) AS token, t.symbol, hex(s.maker) AS first_buyer,
       s.block_time, s.amount_quote
  FROM flap_curve_swaps s
  JOIN flap_tokens t ON t.address = s.token
 WHERE t.created_at >= now() - interval '24 hours'
 ORDER BY s.token, s.block_time ASC, s.log_index ASC
 LIMIT 50;
```

## Indexer health

```sql
SELECT last_block,
       now() - updated_at AS since_last_move,
       sweeps, logs_read, swaps_written, swaps_skipped
  FROM indexer_state WHERE id = 1;
```

The cursor moves every few seconds. `swaps_skipped` grows on swaps of tokens that are not
in the database — that is normal (the "no token, no trade" rule).

## What takes up space

```sql
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(oid)) AS total,
       pg_size_pretty(pg_indexes_size(oid)) AS indexes,
       GREATEST(reltuples, 0)::BIGINT AS estimated_rows
  FROM pg_class
 WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace
 ORDER BY pg_total_relation_size(oid) DESC;
```
