# @pipeworx/hsa-singapore

Medicines registered in Singapore — the Health Sciences Authority (HSA) official Listing of Registered Therapeutic Products (~5,500 products), live from data.gov.sg: SIN licence number, product name, licence holder, approval date, forensic classification, ATC code, dosage form, route, manufacturers, country of manufacture, active ingredients with strengths. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `hsa_sg_search(query?, active_ingredient?, atc_code?, holder?, manufacturer?, country?, forensic_classification?, route?, dosage_form?, approved_from?, approved_to?, limit?, offset?)` — search/filter with real offset+limit pagination and a `total_matches` count, newest approval first. Answers "is X approved in Singapore", "which Singapore-registered products contain Y", "what does company Z hold a licence for".
- `hsa_sg_product(licence_no | name)` — the full registration record for one product, by SIN licence number (exact) or name (exact, else substring, with `match_type` marked). The listing has no indication text, so this returns `indication_text: null` with the authoritative listing and HSA infosearch URLs rather than a paraphrase.
- `hsa_sg_recent_changes(since, limit?)` — products newly registered on or after a date, from the approval date. New registrations only (`removals_visible: false`, see below). Returns `comparable: false, reason: "since_after_published_data"` rather than a false zero when `since` is past the published data.
- `hsa_sg_dataset_info()` — row count, source last-updated date, earliest/latest approval date, licence, fields present and absent, and distinct values with counts for forensic classification, country of manufacture, route and dosage form (exact spellings for the filters).

Every call rejects argument names it does not declare (`user_error: … does not recognise argument(s)`) instead of silently dropping them.

## Auth

Keyless.

## Data sources

- <https://data.gov.sg/datasets/d_767279312753558cbf19d48344577084/view> — "Listing of Registered Therapeutic Products", published by HSA. Read via `https://data.gov.sg/api/action/datastore_search?resource_id=d_767279312753558cbf19d48344577084&limit=10000` (whole listing in one request, 5,511 rows / ~1.3 MB on 2026-09-26) and the freshness stamp from `https://api-production.data.gov.sg/v2/public/api/datasets/d_767279312753558cbf19d48344577084/metadata` (`lastUpdatedAt`, 2026-08-07 at build time).
- <https://www.hsa.gov.sg/e-services/infosearch> — HSA's own product search, where the approved package insert lives. No per-product URL, and it 403s non-browser clients, so the pack links to it rather than reading it.

### Licence

Singapore Open Data Licence v1.0 (<https://data.gov.sg/open-data-licence>): *"You can use, access, download, copy, distribute, transmit, modify and adapt the datasets … whether commercially or non-commercially"*, worldwide and royalty-free, with a conspicuous attribution notice required. Every response carries that notice in `attribution` / `license` / `license_note` via `attachLicense`.

### Gotchas

- **Rate limit.** Keyless data.gov.sg callers get throttled after a couple of requests in quick succession — sometimes as HTTP 429, sometimes as a 200 whose body is `{"code":24,"name":"TOO_MANY_REQUESTS"}`. The pack fetches the whole listing once per isolate (6h cache) and serves a stale copy rather than failing if a refresh is throttled; with no copy held it returns `{found:false, reason:"upstream_throttled"}`.
- **`Approvaldate` is D/M/YYYY** (`19/11/1991`), not M/D. Parsed to ISO.
- **Multi-valued cells use `&&`.** `Activeingredients` and `Strength` pair up by position (`CAMPHOR&&COAL TAR SOLUTION…` / `2% w/w&&10% w/w…`); manufacturer, country, route and ATC code can be multi-valued too. Split into arrays; `CANADA&&CANADA` collapses to one country.
- **No indication text, no status, no cancellation date.** The listing holds only currently registered products. `recent_changes` therefore reports new registrations only; a cancelled product simply disappears. Say so rather than implying a full diff.
- **Therapeutic products only.** Chinese proprietary medicines are a separate HSA listing — the `tcm-products` pack. Medical devices and cosmetics are separate data.gov.sg datasets again.
- The listing is refreshed a few times a year; `data_as_of` on every response is the source's own `lastUpdatedAt`, and approvals after it are not published yet.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "hsa-singapore": {
      "url": "https://gateway.pipeworx.io/hsa-singapore/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/hsa-singapore/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/hsa_sg_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"metformin","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/hsa_sg_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-hsa-singapore` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/hsa-singapore/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "hsa-singapore": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-hsa-singapore"]
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
ask_pipeworx({ question: "your question about Hsa Singapore data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
