---
name: Buy poppi with buyer approval
description: Take a poppi purchase from cart to completed order over MCP, honoring the idempotency key on completion and the buyer-approval invariant on payment.
api: poppi-agentic-commerce
operations:
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-26'
method: generated
source: mcp/poppi-mcp-tools.json (probed tools/list), https://drinkpoppi.com/agents.md, https://drinkpoppi.com/policies/refund-policy
---

# Buy poppi with buyer approval

**Endpoint:** `POST https://drinkpoppi.com/api/ucp/mcp` — anonymous JSON-RPC 2.0.
Every tool requires `meta.ucp-agent.profile`.

## The invariant

> "Checkout requires human approval. Agents must not complete payment without explicit buyer consent."
> — https://drinkpoppi.com/agents.md

If you cannot get contemporaneous buyer approval at the moment of payment, do not call
`complete_checkout`. poppi's own instructions say to route the purchase through Shop Pay via the Shopify
Shop skill (`https://shop.app/SKILL.md`) instead.

## Steps

1. **Discover.** `GET /.well-known/ucp` — confirm `dev.ucp.shopping.checkout` and
   `dev.ucp.shopping.fulfillment` at your protocol version.
2. **Find the items.** `search_catalog` / `get_product` (see the catalog skill).
3. **Cart.** `create_cart` with the line items; `update_cart` to change quantities; `get_cart` to re-read.
4. **Checkout.** `create_checkout` returns line items, totals, taxes and any applicable discounts. This
   step prices the purchase — it does **not** charge.
5. **Fulfill.** `update_checkout` to set the shipping address and method. This store ships to a **single
   destination only** (`allows_multi_destination.shipping: false`) and the only allowed method combination
   is `[shipping]` — do not attempt a split or a pickup leg.
6. **Show the buyer the real number.** Convert minor units to major units first.
7. **Complete — only after approval.** `complete_checkout` requires `meta.idempotency-key` alongside
   `meta.ucp-agent.profile` and the checkout `id`. Generate one key per purchase intent and reuse it on any
   retry so a network failure cannot double-charge. The response carries the order ID and Thank You Page
   URL, or the errors encountered.
8. **Confirm.** `get_order` with the returned order ID.

## Reversibility — read this before step 7

- **Before completion:** `cancel_checkout` and `cancel_cart` both exist and reverse the in-progress
  purchase. poppi states **no time window** for either, so do not promise the buyer one.
- **After completion: there is no programmatic reversal.** No refund, void or return tool exists in this
  tool set. poppi's refund policy states it does *not* accept returns for beverage products bought from
  the website; damaged or broken shipments are handled by emailing Info@motherbeverage.com, with no stated
  window. https://drinkpoppi.com/policies/refund-policy

Treat `complete_checkout` as a one-way door and say so to the buyer when you ask for approval.

## Payment handlers advertised

Google Pay (`com.google.pay`), Shopify card (`dev.shopify.card`), Shop Pay (`dev.shopify.shop_pay`) —
cards accepted: Visa, Mastercard, American Express, Discover, Diners Club.
