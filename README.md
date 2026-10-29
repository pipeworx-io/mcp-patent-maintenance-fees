# @pipeworx/patent-maintenance-fees

Tell whether a US patent is still in force, or has lapsed for non-payment of
maintenance fees — the one question the `patents` pack cannot answer, because
its prosecution-event data has no fee field at all.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `get_patent_maintenance_status(patent_number)` — every recorded
  maintenance-fee event for one patent (payments, surcharges, expiration for
  non-payment, reinstatement, entity-status changes), a computed `status`,
  and a 4th/8th/12th-year fee-window breakdown (due date, grace-period end,
  whether paid). Design (`D`-prefixed) and plant (`PP`-prefixed) patents are
  statutorily exempt from maintenance fees and report
  `subject_to_maintenance_fees:false` instead of a bare no-match.
- `search_maintenance_fee_events(event_code?, patent_number_prefix?,
  date_from?, date_to?, limit?)` — an exact `total_matching` count first,
  then a capped, newest-first sample of the matching events. Useful for
  questions like "how many patents expired for non-payment last quarter" —
  pass `event_code: "EXP."` and a date range.

## Auth

Keyless.

## Data source

USPTO, Patent Maintenance Fee Events — the Open Data Portal's bulk-data
product recording every maintenance-fee event for US patents granted
1981-09-01 to present. US federal data, public domain.

## Why this is a local-copy pack, not a live proxy

Checked first, per CLAUDE.md's local-copy rule: the ODP's Patent File
Wrapper search/get-by-number API — the one `mcps/patents` already calls —
returns prosecution events (`eventDataBag`: filing, publication, grant,
office actions) but has **no maintenance-fee field at all**. Maintenance-fee
history is published only as the "PTMNFEE2" bulk product, a single
weekly-cumulative file, with no per-patent query endpoint. The legacy keyless
host this product used to publish from (`bulkdata.uspto.gov`) no longer
resolves in DNS as of 2026-10-17, confirmed against two independent public
resolvers (NXDOMAIN/NODATA, not a transient block) — the product is
reachable today only through ODP's own authenticated bulk-data API, which
answers every `/api/v1/*` path 401/403 without an API key (the same
credential type as the `patents` pack's `PLATFORM_USPTO_KEY`). A bulk file
with no per-record API is exactly the case CLAUDE.md's sourcing rule calls
for a refreshed backing store rather than a per-call upstream fetch.

This pack reads a Supabase mirror (`uspto_maintenance_fee_events`, schema in
`supabase/migrations/260_uspto_maintenance_fee_events.sql`), loaded by
`scripts/ingest-uspto-maintenance-fees.mjs`
(`.github/workflows/uspto-maintenance-fees-refresh.yml`, dispatch-only — see
below). Every response carries `data_as_of` from the mirror's own
`loaded_at`.

### Not yet provisioned (as of the commit that added this pack)

Two things still need to happen before `uspto_maintenance_fee_events` has any
rows, and both are outside what a commit-only build session can do:

1. **An ODP API key as the `USPTO_ODP_API_KEY` repo secret.** Get one free at
   <https://data.uspto.gov/myodp> (same credential type the gateway already
   holds server-side as `PLATFORM_USPTO_KEY` for the `patents` pack — stored
   again here because the refresh job runs outside the gateway Worker).
2. **The first dispatch of `uspto-maintenance-fees-refresh.yml`.** The
   ingest script's fetch/discovery step (finding and downloading the latest
   `MaintFeeEvents_*.zip` via the ODP bulk-data API) follows the documented
   shape of that API but has **never run against the live endpoint** — no
   key was available in the build session that wrote it. It fails loudly and
   names the actual response shape if that shape doesn't match, rather than
   mis-parsing silently; see the script's header. The fixed-width record
   layout it parses afterward (57 bytes/record, USPTO's own documented
   column offsets) is independently corroborated and covered by unit tests
   in `scripts/__tests__/ingest-uspto-maintenance-fees.test.mjs`.

Until both land, every tool call fails loudly with a `dataset_unavailable`
error rather than silently reporting every patent as having no fee
events — see "Empty-mirror guard" below.

### Scale and load posture

The full file is ~27M event rows (one row per event, not per patent) as of
2026-10. The refresh workflow does a full `TRUNCATE` + `psql \copy` reload
every run — the file is cumulative, so reprocessing it in full reproduces
the correct complete history every time, the same "no delta logic to get
subtly wrong" posture as `scripts/ingest-faa-aircraft-registry.mjs`. Because
a 27M-row reload is a big load, the workflow is `workflow_dispatch` only
(same shape as `sec-13f-load.yml`), not a cron — the first backfill should be
run as a monitored one-off with a Supabase latency/`PGRST002` check before
and after, per Bruce's 2026-09-30 "supa stagger" ruling for SEC 13F.

## Empty-mirror guard

Same silent-zero class fixed for `faa_reg_aircraft` (fleet #2790): an empty
backing table still answers REST queries successfully with zero rows, so a
naive read would report every patent as having no maintenance-fee events —
indistinguishable from a genuine miss against good data. Both tools check
for at least one loaded row first and throw a loud `dataset_unavailable`
error naming the real condition instead.

## Scope — what this pack does NOT determine

Maintenance-fee lapse is **one** way a patent can stop being enforceable, not
the only one. This pack does not determine:

- **Full-term (20-years-from-filing) expiration.** A patent with a perfect
  fee-payment record still expires at the end of its statutory term.
- **Invalidity, reexamination outcomes, or litigation status.**
- **Design and plant patents at all** — see below.

Pair with the `patents` pack for prosecution history and the `epo-ops` pack
for non-US patents.

## Design and plant patent exemption

35 U.S.C. 41(b) applies maintenance fees to utility and reissue patents
only. Design (`D`-prefixed) and plant (`PP`-prefixed) patent numbers never
generate a maintenance-fee event and never appear in this dataset.
`get_patent_maintenance_status` detects the prefix and reports
`subject_to_maintenance_fees:false` with an explanation, rather than a bare
"no events found" that would read identically to a real gap.

## Leading zeros — an open normalization question

Whether the file's fixed-width patent-number column keeps or strips leading
zeros has not been verified against a live file sample (no API key was
available while building this pack — see "Not yet provisioned" above).
`get_patent_maintenance_status` queries both the normalized form and the
same value with leading zeros stripped, rather than assuming one.

## Event codes

USPTO's own long-stable maintenance-fee event codes (payments by
window/entity-size/public-law era, surcharges, `EXP.`/`EXPX`
expiration/reinstatement, entity-status changes, petitions) are decoded in
`src/index.ts`'s `EVENT_CODES` table with the raw code always included
alongside the description — an uncommon or future code not in the table
comes back with the raw code and a `null` description rather than a guess.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "patent-maintenance-fees": {
      "url": "https://gateway.pipeworx.io/patent-maintenance-fees/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/patent-maintenance-fees/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/get_patent_maintenance_status \
  -H 'Content-Type: application/json' \
  -d '{"patent_number":"10835610"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/get_patent_maintenance_status`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-patent-maintenance-fees` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/patent-maintenance-fees/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "patent-maintenance-fees": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-patent-maintenance-fees"]
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
ask_pipeworx({ question: "your question about Patent Maintenance Fees data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
