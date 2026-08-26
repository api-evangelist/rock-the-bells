---
name: rock-the-bells-build-cart
description: Build, inspect, modify and abandon a Rock The Bells cart over UCP/MCP. Pre-purchase only — no payment is involved and every step is reversible.
api: rock-the-bells-commerce-mcp
endpoint: https://shop.rockthebells.com/api/ucp/mcp
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - get_product
generated: '2026-08-26'
method: generated
source: mcp/rock-the-bells-mcp-tools.json (live tools/list, 2026-08-26)
---

# Build a Rock The Bells cart

A cart is a pre-purchase construct. Nothing here charges anyone, and every action has a reversal.

## Steps

1. **Resolve the variant.** Call `get_product` and pick the `ProductVariant` matching the buyer's
   size/colour choice. Adding a product rather than a variant is the most common failure.
2. **`create_cart`** — creates the cart. Keep the returned id.
3. **`update_cart`** — one tool covers what the GraphQL surface splits across nine mutations: add,
   change quantity, remove lines, set a note, apply attributes, apply discount or gift-card codes,
   set buyer identity. Send the desired state; do not hunt for a per-field operation.
4. **`get_cart`** — read back totals before you report anything to the buyer. Do not compute totals
   yourself; tax and discount are the store's to calculate.
5. **`cancel_cart`** — abandon it. This is the reversal of `create_cart` and moves no money. The
   docs state no time window on it.

## Reversibility

| Action | Reversal | Window stated? |
|---|---|---|
| `create_cart` | `cancel_cart` | no |
| `update_cart` (add lines) | `update_cart` (remove lines) | no |

Everything at this stage is undoable. That stops being true the moment you move to checkout — see
`rock-the-bells-complete-purchase`.

## Errors

Cart failures arrive as typed `CartUserError` objects with a `code` from `CartErrorCode`
(`INVALID`, `LESS_THAN`, `INVALID_MERCHANDISE_LINE`, `MERCHANDISE_NOT_APPLICABLE`,
`MISSING_DISCOUNT_CODE`, `NOTE_TOO_LONG`, `INVALID_DELIVERY_OPTION`, `PENDING_DELIVERY_GROUPS`,
`INVALID_PAYMENT`, `PAYMENT_METHOD_NOT_SUPPORTED`, and others). On GraphQL these come back inside a
**200** response body — the HTTP status is not the error signal. Full list in
`errors/rock-the-bells-problem-types.yml`.

## Rate limits

No numeric limit is published. The MCP endpoint is rate-limited per IP and you are told to back off
on **429**. On GraphQL, read `extensions.cost.requestedQueryCost` on every response — that is the
real runtime budget signal, and it is returned whether you ask for it or not.
