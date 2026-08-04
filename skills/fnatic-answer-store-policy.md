---
name: Answer a Fnatic Shop policy or content question
description: >-
  Answer shipping, refund, privacy and terms questions about the Fnatic Shop from the
  store's own published policy documents rather than from memory, and reach editorial
  pages and blog content through the Storefront GraphQL API.
api: graphql/fnatic-storefront.graphql
graphql: graphql/fnatic-storefront.graphql
operations: [shop, page, pages, pageByHandle, blog, blogs, blogByHandle, article, articles, menu, metaobject, metaobjects]
generated: '2026-08-04'
method: generated
---

# Answer a Fnatic Shop policy or content question

Fnatic Shop publishes its policies as first-class objects on the `Shop` type. Read them;
do not paraphrase from general knowledge of Shopify stores.

## Steps

1. **Policies.** Query `shop` and select the `ShopPolicy` fields you need:

   ```graphql
   {
     shop {
       name
       refundPolicy      { title handle url body }
       shippingPolicy    { title handle url body }
       privacyPolicy     { title handle url body }
       termsOfService    { title handle url body }
       termsOfSale       { title handle url body }
       legalNotice       { title handle url body }
       contactInformation { title handle url body }
       subscriptionPolicy { title handle url body }
       shipsToCountries
       paymentSettings { currencyCode countryCode enabledPresentmentCurrencies supportedDigitalWallets }
     }
   }
   ```

   Each policy also resolves over plain HTTP at
   `https://shop.fnatic.com/policies/{handle}` — `privacy-policy`, `terms-of-service` and
   `refund-policy` are the three Fnatic names in its own `/agents.md`.

2. **Editorial pages.** `pageByHandle(handle:)` / `pages(first:)` for store pages;
   `blog`, `blogs`, `blogByHandle`, `article`, `articles` for the shop's blog content;
   `menu(handle:)` for the navigation tree; `metaobject`/`metaobjects` for custom
   structured content.

3. **Shipping reality check.** `shop { shipsToCountries }` is the authoritative list.
   `paymentSettings.enabledPresentmentCurrencies` (39 currencies) tells you what a buyer
   can actually be charged in; the shop currency is EUR and the shop country is GB.

## Where NOT to look

- **Consumer support is not on this surface.** Order issues, returns and warranty go to
  the Fnatic help portal at `https://help.fnatic.com/` (Freshdesk), which is HTML only —
  there is no ticketing API.
- **Esports, team, player and membership questions are not in this schema at all.**
  `shop.fnatic.com` is the merchandise store. Team, roster, news and Fnatic ID membership
  content lives on `https://fnatic.com`, a Next.js front end over a Sanity CMS with **no
  public API**. If asked about rosters, fixtures, or membership tiers, say the data is not
  machine-readable and link the human page — do not scrape and do not guess.

## Rules

- Quote the policy `body` you fetched, and cite the `url`. Policies change.
- The store's shipping threshold and promotional language ("Free shipping over 100€") is
  marketing copy on the shop description, not a policy — verify against
  `shippingPolicy.body` before repeating it.

## Errors

GraphQL returns HTTP 200 with an `errors[]` array. A missing page returns `null` data
rather than an error — handle the null. See `errors/fnatic-problem-types.yml`.
