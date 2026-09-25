# @pipeworx/abs-loan-level

US auto loan / auto lease securitization (ABS) trust delinquency — pre-aggregated monthly rollups derived from SEC EDGAR's Reg AB II asset-level disclosure (Form ABS-EE), by trust and by sector.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `abs_trusts(sponsor?, asset_class?)` — list tracked trusts, filterable by sponsor/depositor name or asset class. Use to find the exact trust name the other two tools need.
- `abs_trust_performance(trust, months=12)` — one trust's delinquency-bucket history (loan count, balance, weighted-average APR/FICO per bucket) over its most recent N reporting months.
- `abs_sector_delinquency(asset_class, months=12, originator?)` — sector-wide delinquency by month, aggregated across every tracked trust in one asset class, optionally narrowed to one sponsor/originator.

## Auth

Keyless — callers need no credential of their own. Queries are answered from pre-computed rollups rather than by parsing EX-102 exhibits at request time, so a call returns in milliseconds regardless of how large the underlying filing was.

## Data sources

- <https://www.sec.gov/cgi-bin/browse-edgar> / <https://efts.sec.gov/LATEST/search-index> — EDGAR full-text search, used by the ingest cron (`workers/scraper/src/abs_ee.ts`) to discover ABS-EE filings.
- Each filing's EX-102 exhibit (asset-level XML, `https://www.sec.gov/Archives/edgar/data/<cik>/<accession>/`) — the underlying loan-level source. **We never persist this raw data.** Reg AB II re-discloses the full current pool every month regardless of issuer size — a single filing runs 100-250MB / 30-80k loan rows, and at ~435 auto loan+lease filings/month that's ~157GB/year of Postgres growth, ~150x the single unbatched load that already crashed production once (fleet #172/#173). See `docs/pack-build-queue.md#dd-07-abs-loan-level` for the full measured numbers and fleet #622 (probe) / #666 (build) for the decision trail.

### What this pack does NOT do

- **No loan-level drill-down.** There is no "which trusts hold loans originated by Lender X in a given state" query. That's an inversion over individual loans, which needs the raw data this pack deliberately excludes. `originator` on `abs_sector_delinquency` filters by the **trust's** sponsor/depositor name (most captive-finance auto trusts are effectively single-originator, so this is close to the same answer for the common case) — it is not a per-loan lookup.
- **No CMBS.** ABS-EE also covers commercial mortgage trusts; they're discovered and logged (for coverage visibility) but never downloaded or rolled up. Not in scope for this pack.
- **No RMBS.** Non-agency RMBS is issued 144A-exempt and never files ABS-EE at all — confirmed empirically against the full filing corpus, not assumed.

### Rollup definition (so numbers are reproducible, not a black box)

`delinquency_bucket` is derived from the SEC schema's `currentDelinquencyStatus` (an integer days-past-due count, not a pre-bucketed code — verified against a live 197MB sample) plus `zeroBalanceCode` (a schema-defined enumerated code that differs between auto loan and auto lease — see `workers/scraper/src/abs_ee.ts` header for both code tables) and, for auto loan only, `repossessedIndicator`:

| bucket | meaning |
|---|---|
| `current` | 0 days past due |
| `days_1_29` / `days_30_59` / `days_60_89` / `days_90_plus` | days-past-due range |
| `repossessed` | auto loan only — the schema has no repossession field for leases |
| `charged_off` | `zeroBalanceCode` = the charged-off code for that asset class, or a non-zero charge-off amount |

`wa_apr` is balance-weighted average interest rate — **always `null` for `auto_lease`**, because the SEC lease asset schema has no interest-rate field at all (leases aren't APR loans). `wa_fico_band` is balance-weighted average obligor/lessee credit score (a continuous number despite the "_band" name — the schema field is a raw score, not a pre-bucketed band).

### Coverage grows over time, not instantly

Each EX-102 filing is 100-250MB; the ingest cron processes one filing per ~10 min to stay well inside Cloudflare Worker CPU limits (a 197MB sample parsed in well under a second of CPU — the wall-clock cost is bandwidth, not compute). `abs_trusts.last_seen` shows what's current; a trust or month with no rows yet means the cron hasn't reached those filings, not that they don't exist. `abs_sector_delinquency` and `abs_trust_performance` both return `found: false` with a `hint` (never an empty array pretending to be a complete answer) when there's no data yet for the requested filter.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "abs-loan-level": {
      "url": "https://gateway.pipeworx.io/abs-loan-level/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/abs-loan-level/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/abs_trusts \
  -H 'Content-Type: application/json' \
  -d '{"asset_class":"auto_loan","limit":10}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/abs_trusts`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "abs-loan-level": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-abs-loan-level"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-abs-loan-level
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Abs Loan Level data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
