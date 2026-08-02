---
name: Browse the ALSO catalog
description: Read ALSO's product and collection catalog from ridealso.com without any
  credential, over either the JSON storefront endpoints or the open Storefront GraphQL
  API.
api: openapi/also-storefront-json-openapi.yml
graphql: graphql/also-storefront-2026-07.graphql
operations: [listProducts, getProduct, listCollections, listCollectionProducts, searchStorefront]
graphql_fields: [products, product, collection, collections, search, predictiveSearch,
  productRecommendations, shop]
generated: '2026-08-02'
method: generated
---

# Browse the ALSO catalog

ALSO sells the TM-B modular Class 3 e-bike, top-frame modules, a delivery quad, the
Alpha Wave helmet and accessories from `https://ridealso.com`. The catalog is readable
anonymously.

## Authentication

None. Both the JSON endpoints and the Storefront GraphQL endpoint answered
unauthenticated on 2026-08-02 — no `X-Shopify-Storefront-Access-Token` was required,
and full GraphQL introspection succeeded. Do not send credentials for read traffic.

## Option A — JSON endpoints (simplest)

1. `listProducts` — `GET https://ridealso.com/products.json` returns every published
   product. Each product carries `id`, `title`, `handle`, `body_html`, `product_type`,
   `tags`, `variants[]`, `images[]`, `options[]`.
2. `getProduct` — `GET https://ridealso.com/products/{handle}.json`, e.g. `tm-b`.
   An unknown handle returns **404** with an empty body.
3. `listCollections` — `GET https://ridealso.com/collections.json`.
4. `listCollectionProducts` — `GET https://ridealso.com/collections/{handle}/products.json`,
   e.g. `gear`. **Guard:** an unknown collection handle returns **200 with an empty
   `products` array**, not 404 — check emptiness, not status.
5. `searchStorefront` — `GET https://ridealso.com/search?q={query}&type=product`
   returns an HTML page, not JSON. Prefer GraphQL for machine consumption.

Pagination on these endpoints is `?limit=` (max 250) and `?page=`.

## Option B — Storefront GraphQL (richer)

`POST https://ridealso.com/api/2026-07/graphql.json` with `Content-Type: application/json`.

- `products(first: Int, after: String, query: String, sortKey: ProductSortKey)` — Relay
  cursor connection; read `edges { node { … } }` and `pageInfo { hasNextPage endCursor }`.
- `product(handle: String)` — single product. (`productByHandle` exists but is
  `@deprecated`; use `product`.)
- `collection(handle: String)` and `collections(first: Int)`. (`collectionByHandle` is
  `@deprecated`.)
- `search(query: String!, types: [SearchType!], first: Int)` and
  `predictiveSearch(query: String!)` for type-ahead.
- `productRecommendations(productId: ID!)` for cross-sell.
- `shop { name description primaryDomain { url } paymentSettings { currencyCode
  acceptedCardBrands } shipsToCountries }` for store context.

## Rules

- Every response carries `extensions.cost.requestedQueryCost`. Keep `first` small and
  page rather than requesting large connections.
- Errors arrive in the top-level `errors[]` array; a 200 HTTP status does not mean the
  query succeeded.
- The version segment is a dated path. Confirm the current version with
  `{ publicApiVersions { handle supported displayName } }` before pinning.
