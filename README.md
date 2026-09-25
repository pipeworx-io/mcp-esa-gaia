# @pipeworx/esa-gaia

ESA Gaia DR3 — positions, parallaxes (hence distances), proper motions,
photometry and astrophysical parameters for 1.8 billion stars, queried over the
ESA archive's TAP service.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `gaia_cone_search(ra, dec, radius_arcmin?, max_magnitude?, min_parallax_mas?, limit?)` —
  every star inside a radius of a sky position, nearest first, with distances
  derived from parallax.
- `gaia_source_by_id(source_id)` — one star's full astrometric and photometric record.
- `gaia_adql_query(query, limit?)` — an arbitrary read-only ADQL SELECT against the
  archive, row-capped at 2,000.

## Auth

Keyless — the synchronous TAP endpoint needs no account.

## Data sources

- <https://gea.esac.esa.int/tap-server/tap/sync> — IVOA TAP, ADQL, `FORMAT=json`.
  Main table `gaiadr3.gaia_source`.

### Things the next person would otherwise rediscover

- **TAP answers a REJECTED query with HTTP 200** and a VOTable `<INFO>` element
  carrying the error, so a naive caller books a failure as an empty result. A
  JSON body with no `data` array is therefore treated as an error and the
  `<INFO>` text surfaced.
- **Parallax is in milliarcseconds and can be NEGATIVE.** It is a measurement
  with noise, not a distance: roughly a fifth of faint sources have a negative
  value. `1000/parallax` is a distance in parsecs only when the parallax is
  positive and well determined, so `distance_pc` is null — with a
  `distance_note` saying why — for a negative parallax or a signal-to-noise
  below 5. Inverting it anyway produces a confident-looking number that means
  nothing.
- **`source_id` is a 64-bit integer.** `JSON.parse` has already lost precision
  above 2^53 by the time the body is parsed, so ids are re-rendered as strings
  on the way out and accepted as strings on the way in.
- **`source_id` values are release-specific** — a DR2 id does not resolve in DR3.
- **The anonymous sync endpoint caps results at 2,000 rows and truncates a
  larger `TOP` silently**, so the cap is applied here explicitly rather than
  trusted to the server.
- **Gaia is dense.** A 1-degree cone in the galactic plane is millions of stars;
  `radius_arcmin` defaults to 3 and is capped at 60 for that reason.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "esa-gaia": {
      "url": "https://gateway.pipeworx.io/esa-gaia/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/esa-gaia/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/gaia_cone_search \
  -H 'Content-Type: application/json' \
  -d '{"ra":56.75,"dec":24.1167,"radius_arcmin":3,"max_magnitude":15,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gaia_cone_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "esa-gaia": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-esa-gaia"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-esa-gaia
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Esa Gaia data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
