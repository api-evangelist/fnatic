---
name: Build a Fnatic Shop cart and hand off to checkout
description: >-
  Create a cart on shop.fnatic.com, add variants, set buyer identity and a single delivery
  address, select a delivery option, and hand the buyer a checkout URL — never completing
  payment without contemporaneous human approval.
api: graphql/fnatic-storefront.graphql
graphql: graphql/fnatic-storefront.graphql
mcp: mcp/fnatic-mcp.yml
operations: [cartCreate, cartLinesAdd, cartLinesUpdate, cartLinesRemove, cartBuyerIdentityUpdate, cartDeliveryAddressesAdd, cartSelectedDeliveryOptionsUpdate, cartDiscountCodesUpdate, cartPrepareForCompletion, cartSubmitForCompletion, cartCompletionAttempt, cart]
generated: '2026-08-04'
method: generated
---

# Build a Fnatic Shop cart and hand off to checkout

## The hard rule first

Fnatic states this in its own `/agents.md`:

> **Checkout requires human approval.** Agents must not complete payment without explicit
> buyer consent.

So: build the cart, resolve delivery, then **stop** and hand the buyer `cart.checkoutUrl`
— or, if Fnatic's recommended path is available to you, route the purchase through the
Shop skill at `https://shop.app/SKILL.md`. Do not call `cartSubmitForCompletion`
autonomously.

## Steps — GraphQL

1. **Create.** `cartCreate(input: { lines: [{ merchandiseId: "gid://shopify/ProductVariant/…", quantity: 1 }] })`.
   Select `cart { id checkoutUrl totalQuantity cost { totalAmount { amount currencyCode } } }`
   and `userErrors { code field message }`.
2. **Edit lines.** `cartLinesAdd(cartId:, lines:)`, `cartLinesUpdate(cartId:, lines:)`,
   `cartLinesRemove(cartId:, lineIds:)`. Read back with `cart(id:)` after every mutation —
   these are **not idempotent** (see below).
3. **Buyer identity.** `cartBuyerIdentityUpdate(cartId:, buyerIdentity: { email, phone, countryCode })`.
   An email is required before completion (`BUYER_IDENTITY_EMAIL_REQUIRED`).
4. **Delivery address.** `cartDeliveryAddressesAdd(cartId:, addresses:)`.
   **Exactly one address.** Fnatic's UCP profile declares
   `allows_multi_destination.shipping = false`; adding a second returns
   `ONLY_ONE_DELIVERY_ADDRESS_CAN_BE_SELECTED` or `TOO_MANY_DELIVERY_ADDRESSES`.
5. **Delivery option.** Read `cart { deliveryGroups(first: 10) { nodes { id deliveryOptions { handle title estimatedCost { amount currencyCode } } } } }`,
   then `cartSelectedDeliveryOptionsUpdate(cartId:, selectedDeliveryOptions:)`.
   Skipping this fails completion with `NO_DELIVERY_GROUP_SELECTED`.
6. **Discounts (optional).** `cartDiscountCodesUpdate(cartId:, discountCodes:)`. Verify
   the code applied by reading `cart { discountCodes { code applicable } }` — an
   inapplicable code is reported as `applicable: false`, not as an error.
7. **Prepare.** `cartPrepareForCompletion(cartId:)` to surface every remaining
   `SubmissionError` (95 codes) *before* the buyer is in front of a payment form.
8. **Hand off.** Give the buyer `cart.checkoutUrl`. Stop here.

## Steps — UCP MCP path

If you hold a UCP agent profile URI, the same flow is
`search_catalog → create_cart → create_checkout → update_checkout → complete_checkout`
against `https://shop.fnatic.com/api/ucp/mcp`. `complete_checkout` still requires
contemporaneous buyer approval. Tool input schemas are only enumerable with an authorized
`tools/list`; see `mcp/fnatic-tool-crosswalk.yml` for how each tool maps onto the GraphQL
mutations above.

## Idempotency — read this before retrying anything

- **Cart mutations are not idempotent.** There is no idempotency key on `cartCreate`,
  `cartLinesAdd`, or any other cart mutation. Repeating `cartLinesAdd` with the same input
  adds the lines **again**. On a timeout, re-read `cart(id:)` and reconcile; never blindly
  retry.
- **`cartSubmitForCompletion` is not idempotent either.** If it times out, poll
  `cartCompletionAttempt(attemptId:)` — do not resubmit.
- **The one idempotent operation on this surface** is
  `shopPayPaymentRequestSessionSubmit(idempotencyKey: String!)`. Reusing a consumed key
  returns `IDEMPOTENCY_KEY_ALREADY_USED`, which means **the charge already happened** —
  treat it as success and go look for the order, never as a reason to mint a new key.

See `conventions/fnatic-conventions.yml`.

## Errors

`CartUserError` (58 codes) on cart mutations, `SubmissionError` (95 codes) on
prepare/submit, `CompletionError` (13 codes) on the completion attempt. The ones you will
actually hit on Fnatic: `ONLY_ONE_DELIVERY_ADDRESS_CAN_BE_SELECTED`,
`ZIP_CODE_NOT_SUPPORTED`, `MERCHANDISE_OUT_OF_STOCK`, `NO_DELIVERY_GROUP_SELECTED`,
`BUYER_IDENTITY_EMAIL_REQUIRED`. Full catalogue at `errors/fnatic-problem-types.yml`.
