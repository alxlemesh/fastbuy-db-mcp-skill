# fastbuy-db skill

A Claude Code skill for the **fastbuy** swap index: a PostgreSQL database of BSC meme-token
trades from Four.meme, Flap and OpenFour — the bonding curve of each and the PancakeSwap
pools they graduate into.

The skill teaches Claude the shape of that database and how to query it without falling into
its four traps: addresses stored as `BYTEA`, `uint256` amounts that must never become
JavaScript numbers, wallets that have to be matched against three columns at once, and
OpenFour, which shares a prefix with Four.meme and nothing else. It ships the full schema
reference and a set of ready-made queries.

Those files are a snapshot, and the database keeps moving: the skill's first instruction is
to call the server's `get_guide`, which serves the same reference as the server currently
holds it, and to believe the server wherever the two disagree.

**The skill is knowledge, not access.** The data lives behind an MCP server that is not part
of this repository; the skill is only useful once Claude can reach that server. Setting that
up is the second half of this page.

## 1. Install the skill

The `claude` commands below are identical on macOS, Linux and Windows.

```bash
claude plugin marketplace add https://github.com/alxlemesh/fastbuy-db-mcp-skill.git
claude plugin install fastbuy-db@fastbuy --scope user
claude plugin list
```

`fastbuy-db@fastbuy` must appear as `✔ enabled`. `--scope user` makes it available in every
project on the machine; remember it, because uninstalling requires the same scope.

If `claude` is not installed yet: `npm install -g @anthropic-ai/claude-code`.

### Updating

A plugin is a copy pinned to the version in its manifest, so a new commit here reaches you
only after:

```bash
claude plugin marketplace update fastbuy
claude plugin update fastbuy-db
```

### Removing

```bash
claude plugin uninstall fastbuy-db@fastbuy --scope user
claude plugin marketplace remove fastbuy
```

## 2. Point Claude at the server

The server speaks MCP over Streamable HTTP and expects a bearer token. You need two things
from whoever runs it:

- the **endpoint** — `http://<host>:2389/mcp` (`2389` is the default port);
- the **token** — a shared secret, checked on every request.

Keep the token out of files. Export it once, ideally from your shell profile:

```bash
export FASTBUY_MCP_TOKEN=…                    # macOS, Linux
setx FASTBUY_MCP_TOKEN "…"                    # Windows, new terminals see it
```

### The quickest way: one command

```bash
claude mcp add --transport http fastbuy-db http://127.0.0.1:2389/mcp \
  -H "Authorization: Bearer $FASTBUY_MCP_TOKEN"
```

Use `-s user` to make the server available in every project. `claude mcp list` should then
show `fastbuy-db … ✔ Connected`.

### Or per project, in a file

Put this in `.mcp.json` at the root of the project you work from:

```json
{
  "mcpServers": {
    "fastbuy-db": {
      "type": "http",
      "url": "http://127.0.0.1:2389/mcp",
      "headers": { "Authorization": "Bearer ${FASTBUY_MCP_TOKEN}" }
    }
  }
}
```

The `${…}` is expanded from your environment, so the file stays free of secrets. A server
declared this way needs one-time approval: run `claude` in that directory and accept it
(until then `claude mcp list` shows it as `⏸ Pending approval`). If the variable is not
exported, `claude mcp list` says so outright: `Missing environment variables:
FASTBUY_MCP_TOKEN`.

### The host

`127.0.0.1` only works when the server runs on your own machine. Otherwise use its address —
`http://10.0.0.5:2389/mcp` — or, better, forward the port over SSH and keep talking to
localhost:

```bash
ssh -L 2389:127.0.0.1:2389 user@server
```

That way the port never has to face the network at all.

### Checking it by hand

```bash
curl -s http://127.0.0.1:2389/health

curl -s -X POST http://127.0.0.1:2389/mcp \
  -H "Authorization: Bearer $FASTBUY_MCP_TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

`/health` needs no token and answers `{"status":"ok",…}`. The second call lists the tools;
a wrong or missing token gives `401` with a `WWW-Authenticate: Bearer` header.

## What the server exposes

Eleven read-only tools. The skill explains when to reach for which:

| Tool               | What it does                                                      |
| ------------------ | ----------------------------------------------------------------- |
| `get_guide`        | The schema reference and recipes as the server holds them now     |
| `get_swaps`        | Swaps by token, wallet, side, quote, pool, period or block range  |
| `get_txs_by_maker` | Every swap of an address — `maker`, `tx_from` and `tx_to` at once |
| `get_tokens`       | Token search and listings                                         |
| `get_token`        | Token card: fields, pools, counters, volumes per quote            |
| `get_wallet`       | Wallet summary                                                    |
| `top_traders`      | Top wallets of a period, by trades or volume                      |
| `list_tables`      | What the database holds and how much it weighs                    |
| `describe_table`   | Columns, types, indexes, foreign keys                             |
| `execute_sql`      | An arbitrary `SELECT`                                             |
| `indexer_status`   | Cursor, lag behind the chain head, table sizes                    |

Every one of them reads. The server connects as a PostgreSQL role that only has `SELECT`,
and each query runs inside a `READ ONLY` transaction, so a write is rejected by the database
rather than by string matching on the query.

## What is in this repository

```
.claude-plugin/marketplace.json   makes this repository a Claude Code marketplace
skills/fastbuy-db/
  SKILL.md                        when to use the skill and the four traps
  references/schema.md            every table, column, type and index
  references/recipes.md           ready-made SQL for the common questions
```

No server code, no addresses, no credentials, no data.
