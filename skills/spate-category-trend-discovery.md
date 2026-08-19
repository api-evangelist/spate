---
name: spate-category-trend-discovery
description: >-
  Resolve a consumer category by name against Spate's taxonomy and pull the
  top-ranked trends inside it for a given market, using Spate's remote MCP
  server. Use when someone asks what is trending in a beauty, wellness, or
  food-and-beverage category.
api: Spate API
surface: mcp
endpoint: https://api.spate.nyc/mcp
operations:
  - find_category
  - top_trends
generated: '2026-08-13'
method: generated
source: mcp/spate-mcp-tools.json
---

# Discover the top trends in a Spate category

Every Spate tool except `find_category` requires a **category UUID**. You will
almost always start from a human phrase ("hair oil", "functional beverages"),
so resolution is a mandatory first hop. Do not guess a UUID — there is no
documented ID format beyond "as a uuid", and a wrong UUID is indistinguishable
from a wrong answer.

## Before you start

- The endpoint is `https://api.spate.nyc/mcp` (JSON-RPC 2.0 over Streamable HTTP).
- `initialize` and `tools/list` answer anonymously. **Every `tools/call`
  requires an OAuth 2.1 bearer token** with scope `mcp`, obtained from
  `https://api.spate.nyc/mcp/oauth/token`.
- Spate does not issue self-serve API keys. Access is provisioned by their
  team and requires an API add-on on a subscription of at least 5 seats. If
  you do not have a token, stop and say so — do not attempt workarounds.

## Steps

### 1. Resolve the category name

Call `find_category`:

```json
{"name": "hair oil"}
```

`name` is the only parameter and it accepts a partial name. If more than one
category comes back, present the candidates to the user rather than picking
one — Spate's taxonomy has five levels and the same word can appear at
several of them.

### 2. Note which level you landed on

The taxonomy runs `cat1` (broadest) through `cat5` (most specific). You need
the level as well as the UUID, because `top_trends` requires both. `cat5` is
the working unit: it is the only level `time_series` accepts and the only
level `top_related_trends` returns.

### 3. Pull the top trends

Call `top_trends`. All five parameters are required:

```json
{
  "category_id": "<uuid from step 1>",
  "level": "cat5",
  "data_source": "popindex",
  "region": "us",
  "limit": "top100"
}
```

- `data_source` accepts **only** `popindex` on this tool — the tool's own
  description says "Only the popularity index is supported for now." Do not
  pass `tiktok` or `reddit` here; they are valid on `time_series` only.
- `region` is one of `us`, `gb`, `fr`, `jp`, `kr`, `global`. There is no
  region-free read, so ask the user which market they mean rather than
  defaulting silently.
- `limit` is a closed enum: `top5`, `top100`, `all`. It is **not** a number
  and there is no cursor, offset or next-page token. If the user wants 20
  rows, request `top100` and truncate client-side.

### 4. Interpret the ranking honestly

Results are ranked by Spate's popularity index, a proprietary composite metric
documented at <https://help.spate.nyc/en/article/the-popularity-index>. It is
a demand signal, not a sales figure. Say which region and which index the
numbers came from whenever you report them.

## Error handling

Errors arrive as JSON-RPC error objects **inside an HTTP 200**. Never route on
HTTP status alone.

| code | meaning | what to do |
|---|---|---|
| `-32001` | `Missing Authorization: Bearer token` | Not retriable. Obtain a token; do not retry the call unchanged. |
| `-32601` | `Method not found` | You called a method the server does not implement. Only `initialize`, `tools/list` and `tools/call` exist — there are no resources or prompts. |

No rate-limit headers are returned and no limits are published, so pace
yourself conservatively and back off on any repeated failure.

## Do not

- Do not invent category UUIDs or brand UUIDs.
- Do not report Instagram or Google figures from this API. Spate tracks them
  in its dashboard, but the MCP `data_source` enum exposes only `popindex`,
  `tiktok` and `reddit`.
- Do not treat `limit: "all"` as unbounded pagination — there is no
  continuation mechanism if the result set exceeds what it returns.
