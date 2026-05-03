# Commerce Kit

## What is Commerce Kit?

Commerce Kit is an open-source, TypeScript-native commerce library project for teams that want a headless commerce engine as code. The current docs define the intended v1 shape for catalog, orders, payments, pricing, tax, shipping, delivery, and the order lifecycle.

## Why it exists

Most commerce solutions bundle platform choices together. Commerce Kit is intended to make the commerce domain installable as a library instead.

## Status

Commerce Kit is currently in the architecture and v1 definition stage. The core direction and boundaries are documented, but the project is not yet a finished production release.

## Quick example

This reflects a proposed v1 usage shape. The names below are illustrative, not final shipped APIs.

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { moyasarPayments } from "@commerce-kit/moyasar"
import { aramexFulfillment } from "@commerce-kit/aramex"
import { marsoolFulfillment } from "@commerce-kit/marsool"
import { couponsPlugin } from "@commerce-kit/coupons"

createCommerce({
  database: drizzleAdapter({ db, provider: "postgres" }),
  payments: [moyasarPayments({ adapterId: "moyasar-main" })],
  fulfillment: [aramexFulfillment(), marsoolFulfillment()],
  plugins: [couponsPlugin()],
})
```

Start with the core store shape above. If you later need a marketplace, install a marketplace plugin that adds vendor-owned domain models, routes, and workflows without making them part of the base setup.

## Documentation

- Start here: [Product overview](./docs/product-overview.md)
- Then: [Getting started example](./docs/examples/getting-started.md)
- Technical reference: [Architecture index](./docs/architecture/00-index.md)
- Competitor research: [API DX research](./docs/competitors/api-dx-research.md)

## Planned v1 scope

Planned v1 focuses on the core engine for a straightforward store: products and variants, cart and checkout flow, orders and lifecycle management, payments, pricing, tax and coupon rules, and unified fulfillment integrations for shipping, local delivery, pickup, or digital delivery. Marketplace capabilities are an opt-in plugin expansion when a business is ready for them.

## Principles

- Library, not platform
- Guest in your stack
- Plugin-first extension model
- Simple-store-first core
- Marketplace as an opt-in expansion
- Easy migration path into marketplace complexity when needed
- Integer minor-unit money rules
- Clear type and runtime boundaries

## What's not included

Commerce Kit does not include storefront UI, authentication, CMS, admin dashboards, analytics, or other non-commerce infrastructure.

## Next

Start with the product overview for the repo-level picture, then the getting-started example for the intended app shape, then the architecture index for the canonical technical reference.
