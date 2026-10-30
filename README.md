# @pipeworx/china-commerce

China's outward commercial activity — outward direct investment, overseas
contracted engineering, overseas labour dispatch, and utilised inbound FDI —
as monthly year-to-date cumulative national totals from the Ministry of
Commerce's data centre.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1861+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `mofcom_odi({ start_period?, end_period?, limit? })` — non-financial
  outward direct investment (对外直接投资): cumulative amount (USD 100M),
  number of enterprises, count of countries/regions reached, and YoY %.
- `mofcom_overseas_contracting({ start_period?, end_period?, limit? })` —
  overseas contracted engineering (对外承包工程): cumulative newly-signed
  contract value and completed turnover, both USD 100M, each with YoY %.
- `mofcom_labor_cooperation({ start_period?, end_period?, limit? })` —
  overseas labour dispatch (对外劳务合作): cumulative workers dispatched
  (10,000-person units), split by contracted-engineering vs labour-cooperation
  route, plus the YoY change (an absolute figure, not a percent).
- `mofcom_fdi_utilised({ start_period?, end_period?, limit? })` — utilised
  inbound FDI (吸收外商直接投资): cumulative new FDI-enterprise count and, for
  periods before mid-2023, the cumulative dollar amount (USD 100M), each with
  YoY %.
- `mofcom_series_list()` — the four series above with their Chinese/English
  names, units, and current earliest/latest period — a cheap discovery call.

`start_period`/`end_period` accept `YYYYMM` or `YYYY-MM`; omit both for full
history. `limit` caps how many of the matched rows come back (newest first,
default 24, max 500).

**Every row is a year-to-date cumulative figure**, not a standalone monthly
flow — China's commerce ministry reports these as Jan-through-month totals
that reset every January. Each tool's response restates this in its `note`
field so a caller does not mistake e.g. September's cumulative total for
September alone.

**No per-country/region breakdown is available.** The "count of
countries/regions" field in `mofcom_odi` is exactly that — a count — not a
split by destination. A separate MOFCOM "country statistics" section exists
but covers unrelated macro indicators (GDP, CPI, trade balance) rather than
ODI/contracting/labour by destination, and is out of scope for this pack.

## Auth

Keyless. No cookie or session — just a same-site `Referer` header, which the
pack sends on every call.

## Data sources

- Ministry of Commerce of the People's Republic of China (中华人民共和国商务部),
  商务数据中心 (the commerce data centre) — 非金融类对外直接投资统计 (outward
  direct investment), 对外承包工程统计 (overseas contracted engineering),
  对外劳务合作统计 (overseas labour dispatch), and 吸收外商直接投资月报
  (monthly utilised inbound FDI) series.

Found via the data centre's own page-loader scripts, not curl-guessing — a
direct API-path probe 404s, and the real calls only surface in each page's
`<slug>.js`. All four are keyless `POST` endpoints with no cookie/session,
just a `Referer` header, and all accept an optional start/end period filter.

The utilised-FDI series (`mofcom_fdi_utilised`) stopped carrying the monetary
figure through this feed around mid-2023 — only the new-enterprise count has
kept reporting since; the pack returns `actual_fdi_usd_100m: null` for those
periods rather than inventing a number or dropping the row.

ODI history starts 2003, overseas contracting starts 1979, labour cooperation
starts 2012, and utilised-FDI starts 1983 — ranges confirmed live via
`mofcom_series_list`, not assumed.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "china-commerce": {
      "url": "https://gateway.pipeworx.io/china-commerce/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/china-commerce/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1861+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/mofcom_odi \
  -H 'Content-Type: application/json' \
  -d '{"start_period":"2026-01","end_period":"2026-09"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/mofcom_odi`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-china-commerce` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/china-commerce/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "china-commerce": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-china-commerce"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about China Commerce data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
