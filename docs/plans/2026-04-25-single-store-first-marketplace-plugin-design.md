# Single-Store-First Marketplace Plugin Design

**Date:** 2026-04-25
**Status:** Approved

## Goal

Rewrite the current Commerce Kit docs so the architecture clearly presents a simple-store-first core and a first-class marketplace plugin expansion path, replacing the current same-schema/no-migration positioning with an easy-migration story.

## Decisions

### 1. Product stance

- simple store is the core default
- marketplace is part of the overall vision now
- marketplace exists as a plugin-based expansion
- migration is acceptable, but should be easy and tooled
- versioning can be ignored for now in favor of clear architectural boundaries

### 2. Core vs marketplace boundary

**Core owns:**

- products
- variants
- orders
- order items
- payments
- pricing
- tax rules
- shipping/delivery adapter seams
- lifecycle and audit
- framework adapters
- client
- CLI and migration tooling

**Marketplace plugin owns:**

- `vendor`
- `vendorOrder`
- `product.vendorId`
- vendor-facing APIs and routes
- order splitting by vendor
- payouts, commissions, onboarding, and related marketplace workflows
- migration path from simple-store data to marketplace-owned data

### 3. Promise rewrite

Replace:

- same schema from day one
- no migration
- mode flip from single-vendor to multi-vendor

With:

- simple stores stay simple
- marketplace is first-class but opt-in
- migration to marketplace should be easy

### 4. Docs rewrite scope

The rewrite is architectural, not cosmetic. It should:

- remove `mode` as a base-core assumption
- remove vendor concepts from the base core data model
- introduce a dedicated marketplace plugin doc
- replace mode-gating language with plugin-gating where appropriate
- update product-facing docs and examples so they match the new boundary

### 5. New architecture doc

Add:

- `docs/architecture/45-marketplace-plugin.md`

This doc owns:

- vendor model
- `vendorOrder`
- ownership migration
- marketplace routes/APIs
- marketplace plugin boundaries

### 6. Rewrite order

1. product-facing docs first (`README.md`, `docs/product-overview.md`)
2. core architecture docs (`10-core-engine.md`, `20-data-model.md`, `40-plugin-system.md`)
3. new marketplace plugin doc (`45-marketplace-plugin.md`)
4. downstream docs (`60-type-inference.md`, `70-packages-and-monorepo.md`, `80-framework-adapters.md`, `90-cli-and-tooling.md`)
5. examples last (`README.md`, `docs/examples/getting-started.md`)

## Success criteria

When the rewrite is done, the docs should state clearly:

- simple store is the clean default
- marketplace is first-class but plugin-based
- migration is expected to be easy, not nonexistent
- vendor concepts do not exist in the base core unless marketplace is installed
