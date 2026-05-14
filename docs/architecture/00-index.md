# Architecture

## Purpose

This directory is the canonical architecture reference for Commerce Kit. It defines the current approved system boundaries, data model, extension model, package layout, transport layer, and tooling conventions. [`/docs/product-overview.md`](../product-overview.md) remains the separate business overview.

## Core decisions

- Commerce Kit is a library, not a platform.
- Commerce Kit is a guest in the application's stack.
- The core architecture is simple-store-first; advanced capabilities are dormant until declared.
- Tenancy (merchants and branches) is a core capability, not a plugin. Plugins remain tenancy-agnostic through column helpers and a `support` contract.
- Marketplace is a tenancy checkout mode (`tenancy.checkout: 'split'`), not a separate plugin.
- Plugins ship as additive features that never depend on each other.
- Monetary values use integer minor units only.
- Invalid operations must fail clearly at compile time, runtime, or both.

## Quick start

The minimum viable `createCommerce()` for a single-store project. Every other key is optional and resolves to a documented default:

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { moyasar } from "@commerce-kit/moyasar"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [moyasar({ secretKey: env.MOYASAR_SECRET })],
})
```

What this gives you: a single-store engine, no tenancy, no server-side cart, an empty calculation pipeline (`subtotal === total`), no scheduler, no fulfillment, no delivery. Every dormant slot stays dormant; the SDK surface narrows accordingly so methods you cannot use do not appear.

Layer features in only when you need them — see the architecture map below. Each capability is opt-in: declaring `tenancy`, `cart: { persistence: true }`, `deliveries: [...]`, or any plugin lights up that surface and nothing else.

## Architecture map

- [Core engine](./10-core-engine.md) — scope, non-goals, principles, dormant activation, and core responsibilities
- [Tenancy](./12-tenancy.md) — `merchant()` / `branch()` column helpers, hierarchical reads, plugin `support` block, and install-time validation
- [Errors](./15-errors.md) — error class hierarchy, core codes, HTTP response shape, server throwing behavior, client result pattern, and plugin error conventions
- [Data model](./20-data-model.md) — persistence conventions, core entities, tenancy entities, invariants, and the order state machine
- [Cart](./22-cart.md) — client-side cart, auto-recalculation, `orders.calculate`, and server persistence
- [Server SDK](./25-server-sdk.md) — `commerce.*` method signatures, inputs, return types, `withContext` binding, and shared conventions
- [Pricing and calculations](./30-pricing-and-calculations.md) — money rules, pricing pipeline, recalculation timing, and audit guarantees
- [Calculation engine](./35-calculation-engine.md) — named steps, `string[]` pipelines, per-merchant runtime overrides, and the `calculation:resolvePipeline` hook
- [Plugin system](./40-plugin-system.md) — `plugin()` factory, `table()` column helpers, `support` block, keyed `on` handlers, `operations`, and forbidden behavior
- [Hooks](./42-hooks.md) — keyed `on` handler model, wildcards, operation type narrowing, transaction semantics, and `runInBackground`
- [Background tasks](./43-background-tasks.md) — immediate side effects, deferred tasks via scheduler adapter, and recurring tasks via `commerce.tasks`
- [Pricing rules plugin](./44-pricing-rules-plugin.md) — `@commerce-kit/pricing-rules`: `discount` / `tax` / `serviceCharge` entities and the three canonical calculation steps
- [Marketplace mode](./45-marketplace-mode.md) — `tenancy.checkout: 'split'`, order groups, cross-merchant carts, and commission boundary
- [Validation](./47-validation.md) — Standard Schema contract for `operations.input`, fulfillment type `data`, delivery pricing `settings`, and webhook payloads
- [Adapter system](./50-adapter-system.md) — database and provider adapter contracts, dormant activation, and fulfillment model boundaries
- [Fulfillment types](./52-fulfillment-types.md) — open type registry with typed data schemas, plugin and inline contributions, validation rules
- [Delivery adapters](./54-delivery-adapters.md) — `deliveries: []` slot for driver-based dispatch adapters, `DeliveryAdapter` contract, `deliveryMethod` entity, lifecycle and webhooks
- [Delivery pricing](./55-delivery-pricing.md) — `deliveryPricing: []` slot for fee-calculation strategies, built-in strategies (free, flat, distance, tiered, weight, zone), `DistanceProvider` abstraction
- [Type inference](./60-type-inference.md) — `createCommerce()`-driven type flow and compile-time guarantees
- [Packages and monorepo](./70-packages-and-monorepo.md) — tooling stack, package map, build conventions, and release rules
- [Framework adapters](./80-framework-adapters.md) — HTTP exposure, route handlers, request context, webhooks, and client SDK rules
- [CLI and tooling](./90-cli-and-tooling.md) — config conventions, CLI commands, generation, and migration workflow

## Recommended reading order

1. [Core engine](./10-core-engine.md)
2. [Tenancy](./12-tenancy.md)
3. [Errors](./15-errors.md)
4. [Data model](./20-data-model.md)
5. [Cart](./22-cart.md)
6. [Server SDK](./25-server-sdk.md)
7. [Pricing and calculations](./30-pricing-and-calculations.md)
8. [Calculation engine](./35-calculation-engine.md)
9. [Plugin system](./40-plugin-system.md)
10. [Hooks](./42-hooks.md)
11. [Background tasks](./43-background-tasks.md)
12. [Pricing rules plugin](./44-pricing-rules-plugin.md)
13. [Marketplace mode](./45-marketplace-mode.md)
14. [Validation](./47-validation.md)
15. [Adapter system](./50-adapter-system.md)
16. [Fulfillment types](./52-fulfillment-types.md)
17. [Delivery adapters](./54-delivery-adapters.md)
18. [Delivery pricing](./55-delivery-pricing.md)
19. [Type inference](./60-type-inference.md)
20. [Packages and monorepo](./70-packages-and-monorepo.md)
21. [Framework adapters](./80-framework-adapters.md)
22. [CLI and tooling](./90-cli-and-tooling.md)

## Reading rules

- Treat these documents as normative.
- Prefer linking between files over repeating whole sections.
- Preserve current documented decisions unless a later RFC changes them.

## Non-goals

- Business positioning and ecosystem narrative
- Storefront, admin, auth, CMS, analytics, or other non-commerce concerns

For product narrative and business context, see [`/docs/product-overview.md`](../product-overview.md).
