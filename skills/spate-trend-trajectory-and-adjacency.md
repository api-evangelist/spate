---
name: spate-trend-trajectory-and-adjacency
description: >-
  Read a Spate category's trajectory over time across the popularity index,
  TikTok and Reddit, and map the categories adjacent to it along a chosen
  affinity dimension. Use for "is this trend growing or fading?" and "what
  else moves with it?" questions.
api: Spate API
surface: mcp
endpoint: https://api.spate.nyc/mcp
operations:
  - find_category
  - time_series
  - top_related_trends
generated: '2026-08-13'
method: generated
source: mcp/spate-mcp-tools.json
---

# Read a trend's trajectory and its adjacent categories

This skill answers two questions that `top_trends` cannot: **where is this
going** (`time_series`) and **what sits next to it** (`top_related_trends`).

## Before you start

Same access rules as every Spate tool: `https://api.spate.nyc/mcp`, OAuth 2.1
bearer token with scope `mcp`, no self-serve keys. `tools/list` is anonymous;
`tools/call` is not.

## Part A — trajectory over time

### 1. Resolve to a cat5 category

Call `find_category` with the user's phrase. `time_series` accepts
`item_type` of `cat5` **or** `brand` only — no other taxonomy level works. If
`find_category` returns a cat1–cat4 node, you need to drill to a cat5 before
this tool will accept it.

> **Brand caveat.** `item_type: "brand"` is valid, but there is **no public
> tool that resolves a brand name to a brand UUID** — `find_category` searches
> categories. If the user asks about a brand and you have not been given its
> UUID out of band, say the public tool set cannot reach it rather than
> substituting a category.

### 2. Call `time_series`

All four parameters are required:

```json
{
  "item_type": "cat5",
  "item_id": "<uuid>",
  "data_source": "popindex",
  "region": "us"
}
```

`data_source` here accepts `popindex`, `tiktok` **or** `reddit`. Run it once
per source when the question is about divergence — search demand and social
conversation frequently move at different times, which is the whole point of
comparing them.

### 3. Report the series with its provenance

Spate publishes no response schema for this tool, so do not assume a date
granularity or a unit. Read them off the payload and state what you actually
received. Spate's own framing for reading a series is at
<https://help.spate.nyc/en/article/the-trend-lifecycle> and
<https://help.spate.nyc/en/article/volume-vs-growth>.

## Part B — adjacency

### 4. Choose an affinity dimension

`top_related_trends` ranks the cat5 categories related to your category along
one `affinity_group`. There are 44 of them, and the choice IS the question.
The full enum is in `mcp/spate-mcp-tools.json`; the ones that carry most
analytical weight:

| goal | affinity_group |
|---|---|
| What ingredient is driving this? | `ingredients`, `food-ingredients`, `clean-beauty` |
| What problem are people solving? | `concerns`, `benefits` |
| Who is pushing it? | `influencers`, `creator-content`, `experts-and-advisers` |
| Where would it sell? | `retailers`, `purchases`, `location-types` |
| What form does it take? | `product-format`, `product-characteristics`, `packaging` |
| Who is it for? | `demographics`, `skin-type`, `skin-tone`, `hair-type` |
| How do people feel about it? | `sentiment`, `questions` |
| Is it a knock-off wave? | `dupes`, `company` |

Ask the user which lens they want if it is ambiguous. Do not iterate all 44 —
that is 44 billable calls against an undocumented quota.

### 5. Call `top_related_trends`

All five parameters are required:

```json
{
  "category_id": "<uuid>",
  "category_level": "cat5",
  "affinity_group": "ingredients",
  "region": "us",
  "limit": "top100"
}
```

Results come back as cat5 categories ranked by popularity index. Feed any of
them straight back into Part A to check whether the adjacent trend is rising
or already past peak.

## Conventions that apply to both parts

- **Errors are JSON-RPC objects inside HTTP 200.** `-32001` is a missing
  bearer token (not retriable as-is); `-32601` is a method the server does not
  implement.
- **No idempotency key, no request-id header** you can quote in a support
  ticket, and **no rate-limit headers**. All four tools are read-only, so a
  duplicate call is safe, but retries are unpaced — back off manually.
- **No versioning.** The server reports MCP protocol `2024-11-05` and a git
  commit SHA as its version. Do not assume the tool set is stable across
  calls; re-read `tools/list` if a schema rejection surprises you.
