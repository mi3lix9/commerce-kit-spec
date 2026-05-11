# Architecture

## Purpose

This directory is the canonical architecture reference for Commerce Kit. It defines the current approved system boundaries, data model, extension model, package layout, transport layer, and tooling conventions. [`/docs/product-overview.md`](../product-overview.md) remains the separate business overview.

## Core decisions

- Commerce Kit is a library, not a platform.
- Commerce Kit is a guest in the application's stack.
- The core architecture is simple-store-first.
- Major domain expansions ship as plugins.
- Monetary values use integer minor units only.
- Invalid operations must fail clearly at compile time, runtime, or both.

## Architecture map

- [Core engine](./10-core-engine.md) — scope, non-goals, principles, simple-store-first boundaries, money rules, and core responsibilities
- [Data model](./20-data-model.md) — persistence conventions, core simple-store entities, invariants, and the order state machine
- [Cart](./22-cart.md) — client-side cart, auto-recalculation, `orders.calculate`, and server persistence
- [Pricing and calculations](./30-pricing-and-calculations.md) — money rules, pricing pipeline, recalculation timing, and audit guarantees
- [Plugin system](./40-plugin-system.md) — `CommercePlugin`, runtime requirements, hooks, major domain expansion rules, and forbidden behavior
- [Hooks](./42-hooks.md) — app-level `before`/`after` hooks, `createHook` factory, operation type narrowing, and `runInBackground`
- [Background tasks](./43-background-tasks.md) — immediate side effects, deferred tasks via scheduler adapter, and recurring tasks via `commerce.tasks`
- [Marketplace plugin](./45-marketplace-plugin.md) — marketplace-owned schema, product ownership, vendor APIs, order splitting, and migration boundaries
- [Adapter system](./50-adapter-system.md) — database and provider adapter contracts, dormant activation, and fulfillment model boundaries
- [Type inference](./60-type-inference.md) — `createCommerce()`-driven type flow and compile-time guarantees
- [Packages and monorepo](./70-packages-and-monorepo.md) — tooling stack, package map, build conventions, and release rules
- [Framework adapters](./80-framework-adapters.md) — HTTP exposure, route handlers, request context, webhooks, and client SDK rules
- [CLI and tooling](./90-cli-and-tooling.md) — config conventions, CLI commands, generation, and migration workflow

## Recommended reading order

1. [Core engine](./10-core-engine.md)
2. [Data model](./20-data-model.md)
3. [Cart](./22-cart.md)
4. [Pricing and calculations](./30-pricing-and-calculations.md)
5. [Plugin system](./40-plugin-system.md)
6. [Hooks](./42-hooks.md)
7. [Background tasks](./43-background-tasks.md)
8. [Marketplace plugin](./45-marketplace-plugin.md)
9. [Adapter system](./50-adapter-system.md)
10. [Type inference](./60-type-inference.md)
11. [Packages and monorepo](./70-packages-and-monorepo.md)
12. [Framework adapters](./80-framework-adapters.md)
13. [CLI and tooling](./90-cli-and-tooling.md)

## Reading rules

- Treat these documents as normative.
- Prefer linking between files over repeating whole sections.
- Preserve current documented decisions unless a later RFC changes them.

## Non-goals

- Business positioning and ecosystem narrative
- Storefront, admin, auth, CMS, analytics, or other non-commerce concerns

For product narrative and business context, see [`/docs/product-overview.md`](../product-overview.md).
