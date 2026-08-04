---
name: Authenticate a Fnatic Shop customer and read their orders
description: >-
  Sign a shopper in to the Fnatic Shop, then read their profile, addresses and order
  history — via the OIDC customer-account flow or the legacy Storefront customer access
  token.
api: graphql/fnatic-storefront.graphql
graphql: graphql/fnatic-storefront.graphql
authentication: authentication/fnatic-authentication.yml
scopes: scopes/fnatic-scopes.yml
operations: [customerAccessTokenCreate, customerAccessTokenRenew, customerAccessTokenDelete, customer, customerCreate, customerUpdate, customerRecover, customerAddressCreate, customerAddressUpdate, customerAddressDelete, customerDefaultAddressUpdate]
generated: '2026-08-04'
method: generated
---

# Authenticate a Fnatic Shop customer and read their orders

There are two distinct identity surfaces, and they are **not** the same account system.

| Surface | Where | Machine-readable |
|---|---|---|
| Fnatic Shop customer accounts | `shop.fnatic.com` (Shopify) | yes — OIDC discovery + GraphQL |
| Fnatic ID / membership | `fnatic.com/account/*` | **no** — no discovery document, no API |

Only the first is scriptable. Never assume a shopper's Fnatic ID and their shop customer
account are linked.

## Path A — OIDC customer accounts (preferred)

Discovery: `https://shop.fnatic.com/.well-known/openid-configuration` (also served as
RFC 8414 metadata at `/.well-known/oauth-authorization-server`, and as an RFC 9728
protected-resource document at `/.well-known/oauth-protected-resource`).

- issuer: `https://shopify.com/authentication/54359195821`
- authorization endpoint: `https://shopify.com/authentication/54359195821/oauth/authorize`
- token endpoint: `https://shopify.com/authentication/54359195821/oauth/token`
- flow: authorization code + **PKCE S256** (the only `code_challenge_methods_supported`)
- grants: `authorization_code`, `refresh_token`, `urn:ietf:params:oauth:grant-type:jwt-bearer`
- id token: RS256, JWKS at `.../\.well-known/jwks.json`
- scopes: `openid`, `email`, `customer-account-api:full`, `customer-account-mcp-api:full`

Request the narrowest scope that does the job. `customer-account-mcp-api:full` unlocks the
authenticated per-shopper MCP surface — distinct from the store-level UCP MCP endpoint,
which is gated on an agent profile, not a shopper token. Bearer tokens go in the
`Authorization` header (`bearer_methods_supported: ["header"]`).

## Path B — legacy Storefront customer access token

Entirely inside the GraphQL schema on `https://shop.fnatic.com/api/2026-04/graphql.json`:

1. `customerAccessTokenCreate(input: { email, password })` → `customerAccessToken { accessToken expiresAt }`.
2. Read with `customer(customerAccessToken: $token)` — select
   `id firstName lastName email phone acceptsMarketing
    defaultAddress { … }
    addresses(first: 10) { nodes { id address1 city zip country } }
    orders(first: 20) { nodes { id orderNumber name processedAt financialStatus fulfillmentStatus
      currentTotalPrice { amount currencyCode }
      lineItems(first: 50) { nodes { title quantity variant { id sku title } } }
      successfulFulfillments(first: 5) { trackingCompany trackingInfo { number url } } } }`.
3. Maintain the token with `customerAccessTokenRenew(customerAccessToken:)`; revoke on
   sign-out with `customerAccessTokenDelete(customerAccessToken:)`.
4. Profile and address maintenance: `customerUpdate`, `customerAddressCreate`,
   `customerAddressUpdate`, `customerAddressDelete`, `customerDefaultAddressUpdate`.
5. Recovery: `customerRecover(email:)`, then `customerReset` / `customerResetByUrl`.
   `customerActivate` / `customerActivateByUrl` for invited accounts.
   `customerAccessTokenCreateWithMultipass` exists for SSO but requires a merchant secret.

There is **no root `order` query** — orders are only reachable through
`Customer.orders`, so an order lookup always requires an authenticated shopper.

## Rules

- **Never handle a shopper's password if Path A is available.** Path B's
  `customerAccessTokenCreate` takes a raw password; the OIDC flow does not.
- **Never log, cache, or echo an access token.** Tokens carry `expiresAt` — respect it and
  renew rather than re-prompting.
- Order, address and email data is personal data. Return only what the shopper asked for.

## Errors

`CustomerUserError` (15 codes: `BLANK`, `INVALID`, `TAKEN`, `UNIDENTIFIED_CUSTOMER`,
`CUSTOMER_DISABLED`, `TOKEN_INVALID`, `NOT_FOUND`, …) on every customer mutation. A cart
mutation attempted against a signed-in cart without a token returns
`MISSING_CUSTOMER_ACCESS_TOKEN` on `CartUserError`. See
`errors/fnatic-problem-types.yml`.
