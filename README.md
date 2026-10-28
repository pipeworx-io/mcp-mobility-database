# @pipeworx/mobility-database

The open catalogue of public-transport data feeds maintained by MobilityData —
~4,500 GTFS Schedule feeds and ~1,900 GTFS-Realtime feeds across ~70 countries,
each with the operator's own download URL, licence and bounding box. The
successor to TransitFeeds.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1877+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `mobilitydb_search_feeds(query?, country?, municipality?, provider?, data_type?, include_inactive?, limit?)`
  — find the transit feed for an agency, city or country.
- `mobilitydb_gtfs_rt_feeds(country?, query?, entity_type?, group_by_agency?, limit?)`
  — realtime feeds (vehicle positions / trip updates / service alerts) with the
  schedule feed each is keyed against.
- `mobilitydb_feed(feed_id)` — everything the catalogue records about one feed,
  plus its sibling realtime streams and any superseding entry.

## Auth

Keyless.

MobilityData also runs an authenticated REST API at `api.mobilitydatabase.org`,
which takes a bearer token minted from a refresh token issued on free signup at
<https://mobilitydatabase.org/sign-up>. This pack deliberately does **not** use
it and declares no platform key: the catalogue export carries the same feed
inventory with no credential, and declaring a key env var the gateway does not
hold un-sinks routing toward a tool that could only refuse. What the
authenticated API would add is per-version dataset history and MobilityData's
own validation reports; if that is wanted, `PLATFORM_MOBILITYDB_TOKEN` and the
code that uses it land together in one change.

## Data sources

- <https://files.mobilitydatabase.org/feeds_v2.csv> — the catalogue export
  (~6,475 rows including superseded entries).

## Traps, verified live 2026-09-17

- **It is real CSV, not line-splittable.** Feed names and notes contain commas,
  quotes, CJK text and nested parentheses — `"TC_大有巴士(公總) (Dayou bus (public bus))"`.
  Splitting on `,` puts an operator's contact email in the `data_type` column.
  There is an RFC 4180 parser in `src/index.ts` for exactly this.
- **One agency publishes three realtime rows**, one per `entity_type`: `vp`
  vehicle positions, `tu` trip updates, `sa` service alerts, each with a
  different URL. Counting "realtime feeds" without collapsing on
  `static_reference` triples the agency count.
- **A realtime feed is unusable without its schedule feed.** `static_reference`
  names it; realtime trip ids only resolve against that specific GTFS feed.
- **Some feeds need the CONSUMER's own credential at the transit agency**
  (`urls.authentication_type` 1 or 2, parameter named in
  `urls.api_key_parameter_name`). That credential is between the caller and the
  agency; it is reported so a caller knows before fetching.
- **`redirect.id` marks a superseded entry** that is still present in the export
  and would otherwise look like a live feed.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "mobility-database": {
      "url": "https://gateway.pipeworx.io/mobility-database/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/mobility-database/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1877+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/mobilitydb_search_feeds \
  -H 'Content-Type: application/json' \
  -d '{"query":"BART","country":"US","limit":2}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/mobilitydb_search_feeds`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "mobility-database": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-mobility-database"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-mobility-database
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Mobility Database data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
