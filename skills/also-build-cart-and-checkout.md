---
name: Build a cart and hand off to checkout on ALSO
description: Create a cart, add TM-B line items, attach buyer identity and a delivery
  address, then hand the buyer to ALSO's hosted checkout — using only mutations that
  exist in the captured Storefront GraphQL schema.
graphql: graphql/also-storefront-2026-07.graphql
operations: [cartCreate, cartLinesAdd, cartLinesUpdate, cartLinesRemove, cartBuyerIdentityUpdate,
  cartDeliveryAddressesAdd, cartSelectedDeliveryOptionsUpdate, cartDiscountCodesUpdate,
  cartPrepareForCompletion, cartSubmitForCompletion]
generated: '2026-08-02'
method: generated
---

# Build a cart and hand off to checkout on ALSO

Endpoint: `POST https://ridealso.com/api/2026-07/graphql.json`. No token required.

## Steps

1. **Resolve the variant.** Query `product(handle: "tm-b") { variants(first: 20) { edges
   { node { id title price { amount currencyCode } availableForSale } } } }`. You need
   the variant `id` (a `gid://shopify/ProductVariant/...`), not the product id.
2. **Create the cart.** `cartCreate(input: { lines: [{ merchandiseId: $variantId,
   quantity: 1 }] })`. Read `cart { id checkoutUrl totalQuantity cost { totalAmount {
   amount currencyCode } } }` and `userErrors { code field message }`.
3. **Adjust lines.** `cartLinesAdd(cartId:, lines:)`, `cartLinesUpdate(cartId:, lines:)`,
   `cartLinesRemove(cartId:, lineIds:)`.
4. **Attach the buyer.** `cartBuyerIdentityUpdate(cartId:, buyerIdentity: { email:,
   countryCode: })`. Country code drives pricing and availability.
5. **Delivery.** `cartDeliveryAddressesAdd(cartId:, addresses:)`, then read
   `cart { deliveryGroups(first: 5) { edges { node { deliveryOptions { handle title
   estimatedCost { amount } } } } } }` and choose one with
   `cartSelectedDeliveryOptionsUpdate`. ALSO ships to a large country list — check
   `shop { shipsToCountries }` first.
6. **Discounts (optional).** `cartDiscountCodesUpdate(cartId:, discountCodes: [...])`.
7. **Finish.** Either hand the buyer the `cart.checkoutUrl` (the safe default — ALSO's
   hosted checkout collects payment and consent), or run
   `cartPrepareForCompletion(cartId:)` followed by
   `cartSubmitForCompletion(cartId:, attemptToken:)` and poll
   `cartCompletionAttempt(attemptId:)`.

## Error handling

Every mutation returns a typed error list — check it even on HTTP 200.

- `CartUserError.code` — 60 members, including `INVALID_MERCHANDISE_LINE`,
  `MERCHANDISE_NOT_APPLICABLE`, `MINIMUM_NOT_MET`, `MAXIMUM_EXCEEDED`,
  `INVALID_DELIVERY_OPTION`, `PENDING_DELIVERY_GROUPS`, `ZIP_CODE_NOT_SUPPORTED`,
  `CART_TOO_LARGE`, `SERVICE_UNAVAILABLE`.
- `SubmissionError.code` on `cartSubmitForCompletion` — 98 members, including
  `MERCHANDISE_OUT_OF_STOCK`, `DELIVERY_NO_DELIVERY_AVAILABLE`,
  `PAYMENTS_METHOD_REQUIRED`, `REDIRECT_TO_CHECKOUT_REQUIRED`.
- `CompletionError.code` on the completion attempt — `PAYMENT_CARD_DECLINED`,
  `PAYMENT_INSUFFICIENT_FUNDS`, `PAYMENT_TRANSIENT_ERROR`, `INVENTORY_RESERVATION_ERROR`.
  Treat `PAYMENT_TRANSIENT_ERROR` as retryable; treat the rest as terminal for that
  attempt.

Full catalog: `errors/also-problem-types.yml`.

## Guardrails

- `REDIRECT_TO_CHECKOUT_REQUIRED` means stop and send the buyer to `checkoutUrl`.
- Never complete a payment without contemporaneous buyer approval — ALSO states this
  explicitly in its own `agents.md`.
- `shopPayPaymentRequestSessionSubmit` requires an `idempotencyKey`; reuse the same key
  on retry or you will get `IDEMPOTENCY_KEY_ALREADY_USED`.
