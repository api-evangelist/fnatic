---
name: Find Fnatic gear, apparel and merch
description: >-
  Search the Fnatic Shop catalog and resolve a specific purchasable variant (size,
  colourway) using the anonymous Storefront GraphQL API on shop.fnatic.com.
api: graphql/fnatic-storefront.graphql
graphql: graphql/fnatic-storefront.graphql
operations: [search, products, product, productByHandle, collection, collectionByHandle, predictiveSearch, productTags, productTypes, variantBySelectedOptions]
generated: '2026-08-04'
method: generated
---

# Find Fnatic gear, apparel and merch

Fnatic Shop sells esports hardware (mice, keyboards, headsets, pads), pro apparel and
merchandise. Your job is to get from a buyer's intent to one concrete, in-stock
**variant id** — apparel is size-and-colour variant-heavy, so a product id alone is
never enough to add to a cart.

## Which surface to use

| Need | Surface | Auth |
|---|---|---|
| Full field control, filters, exact variants | GraphQL `https://shop.fnatic.com/api/2026-04/graphql.json` | none — anonymous |
| Agent-driven purchase flow | UCP MCP `https://shop.fnatic.com/api/ucp/mcp` (`search_catalog`) | UCP agent profile URI required |
| Quick read-only browse | `GET /products.json`, `GET /collections.json`, `GET /products/{handle}.json` | none |

The GraphQL endpoint answered full anonymous introspection on 2026-08-04. The UCP MCP
endpoint refuses `tools/list` without an agent profile URI (`-32001
invalid_profile_url`), so if you do not have one, use GraphQL.

## Steps — GraphQL path

1. List candidates with `products(first:, query:, sortKey:, filters:)` or
   `search(query:, types: [PRODUCT], productFilters:)`. Select
   `id handle title productType vendor tags availableForSale
   priceRange { minVariantPrice { amount currencyCode } } requiresSellingPlan`.
2. Narrow by category with `collection(id:)` / `collectionByHandle(handle:)` then
   `products(first:, filters:)`. Use `productTypes(first:)` and `productTags(first:)` to
   discover the real facet values before guessing them.
3. Use `predictiveSearch(query:)` for typeahead-style intent matching.
4. Fetch the detail with `product(id:)` or `productByHandle(handle:)`. Select
   `variants(first: 100) { nodes { id title sku availableForSale quantityAvailable
   price { amount currencyCode } compareAtPrice { amount currencyCode }
   selectedOptions { name value } } }`.
5. When you already know the option values (e.g. Size = L, Colour = Black), call
   `variantBySelectedOptions(selectedOptions: [{name: "Size", value: "L"}, ...])` — it
   resolves straight to one variant id.
6. Wrap the query in `@inContext(country: DE, language: EN)` to localize. The shop's
   currency is EUR and its shop country is GB, but it declares 39 presentment currencies
   and a fixed `shipsToCountries` list — read `shop { shipsToCountries }` rather than
   assuming.

## Rules

- **Check `availableForSale` and `quantityAvailable` before you promise anything.**
  Limited-drop merch sells out; a variant that lists is not a variant that ships.
- **Never add a product id to a cart.** Cart lines reference a `ProductVariant` through
  the `Merchandise` union. Adding an unresolved variant fails with
  `INVALID_MERCHANDISE_LINE`.
- **Never quote a price you did not read from the response** — prices vary by presentment
  currency and by country.
- Confirm the buyer's country ships: an unsupported destination surfaces later as
  `ZIP_CODE_NOT_SUPPORTED` or `DELIVERY_NO_DELIVERY_AVAILABLE`, after the cart is built.

## Errors

GraphQL returns HTTP 200 with an `errors[]` array — check it before reading `data`. Every
response carries `extensions.cost`; the API is query-cost throttled, not header
rate-limited, so back off on rising `throttleStatus` rather than looking for
`X-RateLimit-*`. The MCP endpoint is rate-limited per IP (Fnatic's own `/agents.md`);
back off on 429. Full catalogue at `errors/fnatic-problem-types.yml`.
