---
name: rock-the-bells-browse-catalog
description: Search and read the Rock The Bells merchandise catalog over its public UCP/MCP server — no credential required, no write of any kind.
api: rock-the-bells-commerce-mcp
endpoint: https://shop.rockthebells.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-26'
method: generated
source: mcp/rock-the-bells-mcp-tools.json (live tools/list, 2026-08-26)
---

# Browse the Rock The Bells catalog

Read-only. Nothing in this skill moves money or creates state.

## Before you call anything

Every tool on this server requires a `meta` object, and inside it `ucp-agent.profile` — a URI that
identifies you to the store. It is required by the published schema on all 13 tools. It is
identification, not authentication: no token is issued or checked.

```json
{"meta": {"ucp-agent": {"profile": "https://your-agent.example/profile"}}}
```

Pass `context.address_country` (ISO 3166-1 alpha-2) and `context.currency` when you have them. The
store's own agent instructions say pricing and availability depend on them.

## Steps

1. **`search_catalog`** — free-text or filtered search over the store's products. Use this when the
   buyer's intent is descriptive ("black hoodie", "something from the cruise").
2. **`lookup_catalog`** — batch resolution when you already hold identifiers. Cheaper than looping
   `get_product`.
3. **`get_product`** — full detail for one product, including its variants. A variant, not a
   product, is what enters a cart.

## Reading prices correctly

Money is an integer in the currency's **ISO 4217 minor units** paired with a currency code. Every
tool description on this server states this verbatim.

`{"amount": 2500, "currency": "USD"}` is **$25.00**, not $2,500.

Divide by 100 for two-decimal currencies before you quote a price to a person. Zero-decimal
currencies such as JPY are already whole units. Getting this wrong is a 100x error in a number a
buyer will act on.

## Identifiers

The MCP and GraphQL surfaces use namespaced global ids — `gid://shopify/Product/8728103813180`. The
undocumented public `/products.json` endpoint returns the bare numeric id instead. If you move
between the two you must translate; see `data-model/rock-the-bells-data-model.yml`.

## When to leave this surface

The MCP tools see commerce only. Rock The Bells' five blogs, 24 pages and shop metadata are not
reachable here — query the Storefront GraphQL API at
`https://shop.rockthebells.com/api/2026-07/graphql.json` for those. See
`mcp/rock-the-bells-tool-crosswalk.yml` for what each surface does and does not cover.
