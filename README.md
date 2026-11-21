# @pipeworx/taiwan-fda-drugs

Taiwan's register of drug licences from the Taiwan FDA (Ministry of Health and Welfare): about 66,500 licences, valid, cancelled and revoked, with Chinese and English product names, active ingredient, licence holder, manufacturers and their countries, dosage form, category, issue and expiry dates. Search in Chinese or English.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2020+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `taiwan_fda_drug_search(query, by?, status?, limit?)` finds licences by words in the Chinese or English product name or the active ingredient (`by: "any"`, the default), the product name only, the ingredient only, or the licence holder's or a manufacturer's name. Latin words match the start of a word; Chinese needs two or more characters in order. Results list valid licences first, newest first, each with the status as the register states it (blank for no cancellation, 已註銷 cancelled, 已廢止 revoked) plus an English reading, and the category with an English name. `status` can be `active`, `cancelled`, `revoked` or `inactive` (cancelled or revoked).
- `taiwan_fda_drug_get(id)` the full record by licence number: licence type, cancellation date and reason, dosage form, controlled-drug class, ingredients, holder and tax id, manufacturers with countries, indications (shortened) and packaging.

```
taiwan_fda_drug_search({query: "阿斯匹靈", limit: 5})
  -> 13 matches; valid licences first, then cancelled ones
taiwan_fda_drug_search({query: "aspirin", by: "ingredient", status: "active", limit: 5})
  -> licences with no cancellation stated, English product names
taiwan_fda_drug_get({id: "衛署藥製字第050247號"})
  -> “榮民”痰咳消顆粒, holder 榮民製藥股份有限公司, valid to 2029-07-22
```

Every answer carries `data_as_of`, `source_file_date`, `answered_from` (always `stored_copy`) and an attribution line. A search with no match returns `found: false` with an `empty_reason`; an unknown id returns `found: false, reason: "unknown_id"`.

## Auth

Keyless.

## Data sources

- Taiwan FDA (Food and Drug Administration, Ministry of Health and Welfare), drug licence open data, synchronised by the agency with its licence system weekly. It is published as one bulk open-data file (about 12 MB compressed, 79 MB unpacked, 28 fields, one row per licence and manufacturer), and the agency's online search host does not answer from our network, so a stored copy answers every call and a weekly job refreshes it. There is no live fallback: before the first load, or if the copy cannot be read, both tools return `found: false, reason: "stored_copy_unavailable"` with a plain message, never an empty result. A refresh that would shrink the register by more than 20 percent, or that finds a changed field list, is refused and the previous copy stays.

### Licence note

The dataset is released under the Government Open Data License, Version 1.0 (政府資料開放授權條款－第1版); the national open-data platform's record for the dataset (全部藥品許可證資料集) lists it as free of charge under that licence. Its terms grant users to use the data "不限目的、時間及地域、非專屬、不可撤回、免授權金" (for any purpose, without limit of time or place, non-exclusively, irrevocably and royalty-free), including "重製、散布…編輯、改作" (reproduction, distribution, editing and adaptation), and allow sublicensing. The condition is attribution to the provider, which every answer carries. Redistribution is permitted.

### What the copy holds, and what to watch

- **One record per licence.** Rows repeat per manufacturer (72,000 rows, 66,500 licences); they are merged on the licence number and the manufacturers become a list.
- **Status as stated.** A blank status means the register states no cancellation; it does not mean the licence is in date. An uncancelled licence can be past its expiry (`expiry_passed`), because cancellation is recorded separately, sometimes long after expiry.
- **What is left out.** Street addresses, directions for use, barcodes and the customs approval number are not stored; indications are shortened to about 200 characters and packaging to 120.
- **English readings are ours.** `status_en`, `licence_type_en`, `category_en` and `controlled_class_en` are added by Pipeworx for the values the register uses; a value we have no reading for is returned in Chinese with a null English field.
- **Ingredients** are written by the register in English capitals, sometimes with several ingredients joined by `;`, and are null for about 3,800 licences.
- **A phrase of three or more Chinese characters** is checked against the records after the index lookup (the index holds pairs of characters). For very common phrases only the first 600 candidates are checked and the answer says `total_is_lower_bound`.
- **Not a prescribing reference.** The register says what is licensed, not what is on sale or suitable for a patient. Traditional Chinese medicine products are in a separate register that this pack does not cover.
- **Fields are positional in the stored objects** (see the header of `src/index.ts`, `scripts/lib/taiwan-fda-parse.mjs` and `scripts/lib/drug-store.mjs`); the test suite builds its fixtures with the loader's own modules and reads them through the pack.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "taiwan-fda-drugs": {
      "url": "https://gateway.pipeworx.io/taiwan-fda-drugs/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/taiwan-fda-drugs/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2020+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/taiwan_fda_drug_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"阿斯匹靈","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/taiwan_fda_drug_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-taiwan-fda-drugs` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/taiwan-fda-drugs/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "taiwan-fda-drugs": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-taiwan-fda-drugs"]
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
ask_pipeworx({ question: "your question about Taiwan Fda Drugs data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
