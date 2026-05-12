# Commerce Kit

## What is Commerce Kit?

Commerce Kit is an open-source, TypeScript-native commerce library for teams that want a headless commerce engine as code. The current docs define the intended v1 shape for catalog, orders, payments, pricing, fulfillment, tenancy (merchants and branches), and the order lifecycle.

## Why it exists

Most commerce solutions bundle platform choices together. Commerce Kit is a library, not a platform — it integrates into an existing stack instead of replacing the runtime, ORM, framework, or deployment model.

## Status

Commerce Kit is currently in the architecture and v1 definition stage. The core direction and boundaries are documented, but the project is not yet a finished production release.

## Quick example

This reflects a proposed v1 usage shape. The names below are illustrative, not final shipped APIs.

### Simple store

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { moyasar } from "@commerce-kit/moyasar"
import { aramex } from "@commerce-kit/aramex"
import { couponsPlugin } from "@commerce-kit/coupons"

createCommerce({
  database: drizzleAdapter(db, { schema }),
  payment: {
    moyasar: moyasar({ secretKey: env.MOYASAR_SECRET }),
  },
  fulfillment: {
    aramex: aramex({ apiKey: env.ARAMEX_KEY }),
  },
  plugins: [couponsPlugin()],
})
```

### Multi-merchant SaaS (each merchant has their own storefront)

```ts
createCommerce({
  database: drizzleAdapter(db, { schema }),
  payment: { moyasar: moyasar({ secretKey: env.MOYASAR_SECRET }) },
  tenancy: {
    merchants: true,
    branches: true,
  },
  plugins: [couponsPlugin()],
})
```

### Marketplace (shared storefront, cross-merchant cart)

```ts
createCommerce({
  database: drizzleAdapter(db, { schema }),
  payment: { stripe: stripe({ secretKey: env.STRIPE_SECRET }) },
  tenancy: {
    merchants: true,
    checkout: "split",
  },
})
```

The same `couponsPlugin` works in all three configurations without any change. Plugins are tenancy-agnostic by default.

## Documentation

- Start here: [Product overview](./docs/product-overview.md)
- Then: [Getting started example](./docs/examples/getting-started.md)
- Technical reference: [Architecture index](./docs/architecture/00-index.md)
- Competitor research: [API DX research](./docs/competitors/api-dx-research.md)

## Planned v1 scope

Planned v1 focuses on the core engine for a straightforward store: products and variants, orders and lifecycle management, payments, pricing, an append-only payment ledger, and fulfillment integrations for shipping, local delivery, pickup, or digital delivery.

Tenancy (multi-merchant, multi-branch, marketplace checkout) is a core capability with dormant activation — off by default, declared in `createCommerce()` when needed. Plugins layer on top through a single keyed-handler API and never depend on each other.

## Principles

- Library, not platform
- Guest in your stack
- Simple-store-first core; complexity is opt-in
- Tenancy is core, not a plugin — but dormant until declared
- Plugins are tenancy-agnostic and never depend on each other
- Integer minor-unit money rules
- Clear type and runtime boundaries

## What's not included

Commerce Kit does not include storefront UI, authentication, CMS, admin dashboards, analytics, or other non-commerce infrastructure.

## Next

Start with the product overview for the repo-level picture, then the getting-started example for the intended app shape, then the architecture index for the canonical technical reference.
