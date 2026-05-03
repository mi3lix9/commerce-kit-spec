# Product Overview

Commerce Kit is an open-source TypeScript commerce library: a headless engine teams install into their own backend application instead of adopting as a full platform.

It is aimed first at startups and agencies building custom commerce products on their own backend stack. SaaS teams with embedded commerce needs are a secondary audience.

Commerce Kit is currently in the architecture and v1 definition stage. This document describes the intended product direction, not a finished production release.

## Why It Matters

Commerce Kit is for teams that want to add commerce capabilities without turning that decision into a broader rewrite of how their product is built and operated.

## Who It Is For

Commerce Kit is built primarily for:

- **Startups** building custom commerce experiences that need speed without lock-in
- **Agencies** delivering multiple stores and wanting a reusable commerce foundation across projects
- **SaaS teams** with commerce-adjacent products that need orders, catalog, pricing, or payments as embedded capabilities

It can also be relevant to integration providers, but they are not the primary audience for this overview.

## The Market Problem

For teams building custom commerce applications on modern TypeScript stacks, many existing ecommerce options are still delivered as platforms. They often require adopting the platform's runtime, conventions, and extension model before the team can ship product value.

That creates a few recurring problems:

1. **Lock-in starts early.** Teams spend time adapting to platform constraints instead of building differentiated product experiences.
2. **Growth paths are expensive.** Moving from a simple store to a marketplace often becomes a disruptive expansion project with too much custom infrastructure work.
3. **Integrations are rebuilt repeatedly.** Shipping, payments, tax, and compliance work is frequently reimplemented store by store.
4. **Engineering effort goes to infrastructure.** Teams rebuild the same commerce engine mechanics instead of investing in customer experience and business-specific features.

Commerce Kit exists to make the commerce engine reusable, composable, and maintainable as shared infrastructure.

## What Commerce Kit Is Intended to Cover

The intended v1 scope is a reusable commerce foundation: product catalog, pricing, orders, payments, shipping, tax, and the order lifecycle that connects them.

In product terms, the goal is to reduce repetitive, high-risk commerce work so teams can focus more of their effort on customer experience, business model, and differentiated product features. The intended shape is a reusable engine for the essential mechanics of selling while still leaving application APIs, admin workflows, and surrounding product experience in the team's hands.

## Growth Without Replatforming

Commerce Kit is built for teams that may start with a single-store business today but do not want that choice to limit future growth.

The product direction is flexibility over time: a company should be able to launch with a straightforward store, validate demand, and later expand into marketplace-style operations without changing its overall stack strategy. Commerce Kit keeps the core focused on common store needs first, then adds marketplace capabilities as an opt-in plugin when the business actually needs them.

This is especially valuable for founders and operators who want to reduce strategic risk. They can optimize for the present while preserving room for new vendor relationships, new revenue models, and broader marketplace ambitions later. The migration story is intended to be easier and guided, not denied: expanding into marketplace complexity should aim to feel like a planned upgrade rather than a surprise replatform.

## Plugin Ecosystem Direction

The plugin model is intended to make integrations and feature extensions more reusable across projects. That includes major domain expansions such as marketplace support, which stays first-class in the product vision while remaining opt-in for teams that only need a store.

## Product Boundaries

Commerce Kit is intended to stay focused on the commerce layer and leave the rest of the product stack to the team using it.

That means it does not try to own:

- storefront rendering or UI
- authentication strategy
- CMS/content workflows
- transactional email infrastructure
- analytics/reporting stacks
- a bundled admin system as a required part of adoption

These boundaries are part of the product philosophy, not missing pieces. The intended adoption model is to install Commerce Kit into an existing backend, expose commerce flows through application APIs, and operate catalog, pricing, orders, and fulfillment through application workflows, internal tools, or admin surfaces built to fit the business. The goal is to keep Commerce Kit focused on commerce while letting teams keep the surrounding tools and workflows that already fit their business.

## Bottom Line

Commerce Kit is intended to give teams a reusable commerce foundation without forcing them into a full-system adoption. The product promise is a simpler path to core commerce capabilities and a clearer route from a simple store today to marketplace expansion later through plugins and guided migration.
