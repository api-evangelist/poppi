---
name: Browse and search the poppi catalog
description: Find poppi prebiotic soda products, variants and prices over the store's anonymous MCP endpoint, without starting a purchase.
api: poppi-agentic-commerce
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-26'
method: generated
source: mcp/poppi-mcp-tools.json (probed tools/list), https://drinkpoppi.com/agents.md
---

# Browse and search the poppi catalog

The poppi store exposes a read-only catalog through its Universal Commerce Protocol MCP endpoint. No
credential is required — a `tools/list` call returns 200 anonymously.

**Endpoint:** `POST https://drinkpoppi.com/api/ucp/mcp`
**Content-Type:** `application/json` · **Accept:** `application/json, text/event-stream`

## Before you call

Every tool requires `meta.ucp-agent.profile` — your agent's UCP profile URI. Omit it and the call is
malformed. Pass buyer context (`catalog.context.address_country`, `catalog.context.currency`) so the
prices and availability you get back are the ones the buyer would actually see.

## Steps

1. **Confirm capabilities.** `GET https://drinkpoppi.com/.well-known/ucp` and check that
   `dev.ucp.shopping.catalog.search` is present at the version you speak (currently `2026-04-08`;
   `2026-01-23` is also advertised).
2. **Search.** Call `search_catalog` with `catalog.query` (natural language) and/or `catalog.filters`
   (`categories`, `price.min` / `price.max` in minor units). At least one of query or filters is required.
3. **Page.** Results are paginated and the first page is deliberately short. Take `pagination.cursor` from
   the response and pass it back on the next `search_catalog` call only when the buyer asks for more.
4. **Look up specifics.** `lookup_catalog` resolves several products or variants by identifier in one call;
   `get_product` returns complete detail for a single one.

## Rules that will bite you

- **Prices are integers in ISO 4217 minor units.** `{"amount": 600, "currency": "USD"}` is **$6.00**.
  Divide by 100 for two-decimal currencies before you say a number out loud. JPY and other zero-decimal
  currencies are already whole units.
- **Back off on 429.** The endpoint is rate limited per IP. Watch `shopify-complexity-score-v2` on the
  response to budget how expensive your calls are; there is no published numeric quota.
- **This skill never transacts.** Nothing here charges anyone. Use the buy skill for that, and only with
  contemporaneous buyer approval.

## Read-only alternatives (no MCP)

poppi also publishes plain JSON for agents that only need to read: `GET /products/{handle}.json`,
`GET /collections/{handle}/products.json`, `GET /collections/all`, `GET /search?q={query}&type=product`.
