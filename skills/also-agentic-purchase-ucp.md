---
name: Buy from ALSO as an agent over UCP/MCP
description: Use ALSO's Universal Commerce Protocol MCP endpoint to search the catalog,
  build a cart and drive a buyer-approved checkout, with the agent-profile and
  idempotency requirements the store actually enforces.
mcp: mcp/also-mcp.yml
crosswalk: mcp/also-tool-crosswalk.yml
operations: [search_catalog, lookup_catalog, get_product, create_cart, get_cart, update_cart,
  cancel_cart, create_checkout, get_checkout, update_checkout, complete_checkout, cancel_checkout,
  get_order]
generated: '2026-08-02'
method: generated
---

# Buy from ALSO as an agent over UCP/MCP

## Discover first

`GET https://ridealso.com/.well-known/ucp` (HTTP 200). It returns the merchant profile:
supported UCP versions (`2026-04-08`, `2026-01-23`), the `dev.ucp.shopping` service with
`transport: mcp` and `endpoint: https://ride-also.myshopify.com/api/ucp/mcp`, the
capability set (cart, checkout, fulfillment, discount, order, catalog.search,
catalog.lookup) and the payment handlers (Google Pay, Shopify card, Shop Pay).

Never hardcode the endpoint — the UCP OpenRPC spec requires resolving it from the
discovery profile.

## You must identify yourself

Every call carries `meta.ucp-agent.profile` — a URI resolving to **your** platform's
UCP profile document (HTTP header `UCP-Agent`). Omitting it returns HTTP 422 with
JSON-RPC error `-32001` / `invalid_profile_url`. This was verified live: an anonymous
`tools/list` is refused. There is no bearer token to obtain; the gate is agent identity.

## Flow

1. `search_catalog` — find products matching the buyer's intent. Pass
   `context.address_country` and `context.currency` for accurate pricing and availability.
2. `lookup_catalog` / `get_product` — resolve exact items and variants.
3. `create_cart` → `update_cart` → `get_cart`.
4. `create_checkout` → `update_checkout` (shipping address and method) → `get_checkout`.
5. `complete_checkout` — **only after explicit buyer approval.**
6. `get_order` to confirm.

## Non-negotiables

- **Human approval.** ALSO's `agents.md`: *"Checkout requires human approval. Agents
  must not complete payment without explicit buyer consent."* If you cannot get
  contemporaneous approval, route through the Shop skill at `https://shop.app/SKILL.md`
  instead of completing payment yourself.
- **Idempotency.** Send `meta.idempotency-key` (a UUID, mapped to the HTTP
  `Idempotency-Key` header) on every mutating call and **reuse the same key on retry**.
- **Rate limits.** The MCP endpoint is rate-limited per IP. Back off on 429.
- **Fulfillment constraints** from the discovery profile: `allows_multi_destination.shipping`
  is `false` and the only allowed method combination is `["shipping"]` — do not attempt
  split-destination orders.

## When UCP is not enough

`mcp/also-tool-crosswalk.yml` maps each tool to its GraphQL and REST equivalents on the
same host. Collections browsing, customer-account operations and editorial content have
no UCP tool — use the Storefront GraphQL API for those.
