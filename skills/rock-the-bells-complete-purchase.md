---
name: rock-the-bells-complete-purchase
description: Drive a Rock The Bells checkout to a placed order over UCP/MCP — including the human-approval invariant the store requires before any payment is finalized.
api: rock-the-bells-commerce-mcp
endpoint: https://shop.rockthebells.com/api/ucp/mcp
operations:
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-26'
method: generated
source: >-
  mcp/rock-the-bells-mcp-tools.json (live tools/list, 2026-08-26),
  https://shop.rockthebells.com/robots.txt, https://shop.rockthebells.com/llms.txt,
  https://shop.rockthebells.com/policies/refund-policy
---

# Complete a Rock The Bells purchase

## Read this first — the store's own rule

Rock The Bells states, in **both** `/llms.txt` and `/robots.txt`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically —
> no scripted form fills, browser automation, or end-to-end agent flows that finalize payment
> without an explicit, contemporaneous human approval step.

There is no API credential guarding `complete_checkout`. **Human approval is the access control.**
If you cannot obtain contemporaneous buyer consent at the moment of payment, stop before step 4 and
hand the checkout to the buyer, or route the purchase through the platform shopping skill the store
itself recommends (`https://shop.app/SKILL.md`).

## Steps

1. **`create_checkout`** — from a cart built with `rock-the-bells-build-cart`. Returns line items,
   totals, discounts and taxes. Ids look like `gid://shopify/Checkout/abc123`.
2. **`update_checkout`** — set shipping address and delivery method, then billing and payment
   selection. Country codes are ISO 3166-1 alpha-2. This store ships to a **single destination per
   order** and offers shipping only (no split fulfilment, no pickup combination) — declared in the
   `dev.ucp.shopping.fulfillment` capability config in `/.well-known/ucp`.
3. **`get_checkout`** — read back the final totals and **show them to the buyer**, converted from
   minor units. This is the number they are approving.
4. **Obtain explicit human approval.** Not implied, not previously granted — contemporaneous.
5. **`complete_checkout`** — pass `meta.idempotency-key`. The schema documents it as "an
   idempotency key for completing the checkout". Reuse the same key on any retry. On the GraphQL
   side the equivalent argument is required and a replay is rejected by name with
   `IDEMPOTENCY_KEY_ALREADY_USED`.
6. **`get_order`** — confirm the order exists before telling the buyer it does.

## Failure handling on payment

`CompletionErrorCode` values are returned on completion failure. Exactly one is safe to retry:

- **`PAYMENT_TRANSIENT_ERROR`** — retry, with the same idempotency key.
- `PAYMENT_CARD_DECLINED`, `PAYMENT_INSUFFICIENT_FUNDS`, `PAYMENT_INVALID_CREDIT_CARD`,
  `PAYMENT_INVALID_BILLING_ADDRESS`, `PAYMENT_CALL_ISSUER`, `PAYMENT_AMOUNT_TOO_SMALL`,
  `PAYMENT_INVALID_CURRENCY`, `PAYMENT_GATEWAY_NOT_ENABLED_ERROR`, `INVENTORY_RESERVATION_ERROR` —
  **terminal for this attempt.** They need the buyer, not a retry loop. Report the condition; do not
  re-fire.

## Reversibility — know this before step 5, not after

- **Before completion:** `cancel_checkout` exists as a first-class tool. No stated time window; the
  practical boundary is completion itself.
- **After completion:** there is **no self-service refund API**. The reversal is a **14-day return
  window from order receipt**, stated verbatim in
  <https://shop.rockthebells.com/policies/refund-policy>, and it is executed by a human emailing
  `customercare@rockthebells.com` with the order number, item and reason.
  - Outbound shipping fees are **not** refunded.
  - Items must be unused and in the condition received.
  - Final-sale items cannot be returned, refunded or exchanged.
  - No exchanges — the buyer returns and re-orders.
  - After the warehouse receives the return: up to 5 business days to inspect, then 7–10 business
    days for the refund to land.

You can tell a buyer the window. You cannot action it. Say so plainly before they approve.

## There is no sandbox

No test mode, test card, or simulation surface is published. You cannot rehearse this flow. Every
`complete_checkout` call is real money.
