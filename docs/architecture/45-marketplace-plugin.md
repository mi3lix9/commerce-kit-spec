# Marketplace Plugin

## Purpose

Define the marketplace plugin as the owner of multi-vendor commerce capabilities on top of the simple-store core.

## Non-goals

- redefining the core order lifecycle
- moving base product, variant, order, or payment ownership out of core
- claiming that marketplace expansion happens with no migration

## Core decisions

- Marketplace is a plugin-based expansion, not a base-core mode.
- The marketplace plugin owns vendor topology and related workflows.
- Core remains valid without any marketplace schema or APIs installed.
- Expansion from simple store to marketplace should be easy and tooling-assisted, but it is allowed to add schema and data migrations.
- Marketplace routes are plugin-contributed but exposed only through the core/framework-adapter route mediation path.
- Core order states remain canonical for the customer order; marketplace adds vendor-scoped workflow on top.

## Plugin-owned domain

The marketplace plugin owns:

- `vendor`
- `vendorOrder`
- vendor ownership on products
- vendor-facing APIs and routes
- order splitting by vendor
- payouts, commissions, and onboarding boundaries

### `vendor`

Fields: `id`, `name`, `slug`, `status`, `metadata`, `pluginData`, `createdAt`, `updatedAt`

`vendor` is not part of the core model. It exists only when the marketplace plugin is installed.

### Product ownership

The plugin extends core `product` ownership with a vendor relation such as `product.vendorId`.

That ownership is plugin-defined because the core catalog must remain valid for a simple store with no vendor topology.

Normative rule: when the marketplace plugin is installed, every vendor-owned product must reference exactly one `vendor.id`. Products that are not assigned to a vendor are outside the marketplace surface until onboarding or migration tooling attaches ownership.

### `vendorOrder`

Fields: `id`, `orderId`, `vendorId`, `status`, `subtotal`, `total`, `currency`, `pluginData`, `createdAt`, `updatedAt`

`vendorOrder` represents the plugin-owned split view of a core order for vendor-level operations.

Normative rule: `vendorOrder.orderId` references the canonical core `order.id`, and `vendorOrder.vendorId` references `vendor.id`.

The plugin also assigns each core `orderItem` to exactly one vendor-owned order segment. That relation may be represented through a plugin-owned foreign key such as `orderItem.vendorOrderId` or an equivalent join model, but it must preserve a one-vendor-per-line-item rule after split calculation.

Normative financial split rules:

- `vendorOrder.currency` must equal the parent `order.currency`.
- A core order may contain only one currency; marketplace splitting must not create mixed-currency vendor segments inside a single order.
- `vendorOrder.subtotal` is the sum of the assigned line-item totals before order-level discounts, tax, or fulfillment allocation.
- Any order-level `discountTotal`, `taxTotal`, and `fulfillmentTotal` attributed to marketplace processing must be allocated across `vendorOrder` records by a deterministic plugin rule.
- The plugin must persist enough allocation data or metadata to audit how each vendor segment total was derived.
- The sum of all `vendorOrder.subtotal` values must equal `order.subtotal`.
- The sum of all allocated vendor discounts must equal `order.discountTotal`.
- The sum of all allocated vendor taxes must equal `order.taxTotal`.
- If fulfillment is active, the sum of all allocated vendor fulfillment amounts must equal the corresponding order-level fulfillment total.
- The sum of all `vendorOrder.total` values must equal `order.total`.

## Order splitting boundary

Core owns the canonical customer order.

The marketplace plugin owns:

- splitting a core order into vendor-scoped operational records
- associating line items with the correct vendor-owned order segments
- vendor-level status workflows on `vendorOrder`

This keeps the base order model simple while allowing marketplace workflows to layer on top.

### Lifecycle rule

- Core `order.status` remains the canonical customer-facing lifecycle and source of truth for the financial order record.
- `vendorOrder.status` is the operational lifecycle for each vendor-owned segment.
- The marketplace plugin may register additional plugin states and transitions where vendor workflow must be reflected in Commerce Kit's typed state machine.
- Marketplace must not redefine the meaning of base core order states; it may coordinate them from vendor-level progress and may extend them only through the plugin `states` contract.
- Invalid transitions for plugin-defined marketplace states must be rejected with the same compile-time and runtime guarantees as core order transitions.

### Status aggregation rule

- Mixed vendor progress is allowed: different `vendorOrder` records under the same `order` may be in different valid statuses at the same time.
- `order.status` is not a mirror of any single `vendorOrder.status`.
- The marketplace plugin is responsible for aggregating vendor-level progress into a valid core `order.status` using declared transition rules.
- The core order may advance only when the aggregate marketplace condition for the next core state is satisfied.
- A single vendor segment entering a terminal or exceptional state does not implicitly rewrite the whole order into the same state; aggregation must be explicit and deterministic.

## Vendor APIs and routes

Vendor-facing APIs are defined by the marketplace plugin.

Vendor-facing HTTP routes are contributed by the marketplace plugin but exposed through the core route layer and framework adapters, not through raw plugin-owned router registration.

Examples include:

- vendor onboarding and profile management
- vendor catalog management
- vendor order operations
- vendor payout or commission reporting surfaces

Framework adapters may expose those APIs over HTTP, but the capability boundary belongs to the plugin.

## Payouts, commissions, and onboarding boundaries

The marketplace plugin owns the business boundary for:

- vendor onboarding workflows
- commission calculation inputs and records
- payout orchestration requirements and related integrations

Core may expose an optional payout adapter interface for provider integrations. The marketplace plugin consumes that interface when marketplace payouts are enabled, and defines the vendor-facing contract, commission rules, payout scheduling/orchestration boundary, and related data ownership.

Specialized payout providers or accounting integrations may still be implemented through adapters or subordinate plugins, but the marketplace plugin defines the marketplace-facing contract and data ownership boundary.

## Migration and tooling stance

Commerce Kit does not promise that marketplace expansion requires no migration.

Instead, the promise is:

- the core model starts simple
- marketplace expansion is first-class
- adoption should be guided by tooling and documented migration steps

Expected tooling may include schema generation, backfill helpers, validation checks, and install-time setup flows that attach vendor ownership to existing catalog and order data where needed.

Historical immutable financial data must be preserved. Migration tooling may safely:

- create marketplace-owned tables such as `vendor` and `vendorOrder`
- attach vendor ownership to existing products
- create vendor-owned order segments only for historical orders whose vendor ownership can be established from durable evidence such as pre-existing product ownership mappings, archived catalog records, or explicit operator-supplied mappings
- backfill non-destructive linkage records and operational metadata where the evidence is sufficient

If historical vendor ownership cannot be proven safely, migration tooling must leave those historical orders unsplit, mark them as requiring manual attribution, or scope marketplace splitting to orders created after marketplace adoption.

Migration tooling must not rewrite historical core order totals, payment ledger rows, or existing transition records.

## Relationship to other architecture docs

- [10-core-engine.md](./10-core-engine.md) defines what core excludes.
- [20-data-model.md](./20-data-model.md) defines the simple-store core entities.
- [40-plugin-system.md](./40-plugin-system.md) defines the plugin contract that the marketplace expansion uses.
