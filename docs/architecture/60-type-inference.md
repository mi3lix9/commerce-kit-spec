# Type Inference

## Purpose

This document defines how Commerce Kit derives types from `createCommerce()` and which compile-time guarantees that flow must provide.

Plugin contracts live in [40-plugin-system.md](./40-plugin-system.md). Adapter activation and dormant interfaces live in [50-adapter-system.md](./50-adapter-system.md).

## Non-goals

- Repeating full plugin or adapter contracts
- Defining runtime transport behavior
- Documenting TypeScript patterns unrelated to Commerce Kit inference

## Core decision

`createCommerce()` is the single source of truth for Commerce Kit types. Types flow from configuration into the server instance, route handlers, and client SDK. Developers do not manually assemble or augment Commerce Kit types.

## Type flow

```ts
export const commerce = createCommerce({ ... })

export type Types = InferCommerceTypes<typeof commerce>
type Order = Types["Order"]
type OrderStatus = Types["OrderStatus"]
type Product = Types["Product"]
type ClientApi = Types["ClientApi"]
```

Normative flow:

1. `createCommerce<TConfig>()` captures config as a literal type.
2. Plugin tuples are walked to merge plugin API namespaces.
3. Plugin tuples and activated adapters are walked to build the full order-state union.
4. Installed plugins narrow and expand the server API surface.
5. Activated adapters materialize additional API surface.
6. `createCommerceClient<typeof commerce>()` from `@commerce-kit/client` infers the client from the server instance type.

## Compile-time guarantees

For direct typed usage patterns based on literal config and known property access, Commerce Kit should produce static errors for at least the following cases:

- accessing a plugin API that is not installed
- accessing marketplace APIs when `@commerce-kit/marketplace` is not installed
- accessing `commerce.fulfillment.*` when no fulfillment adapter is registered
- performing an invalid order-status transition for the installed plugin and adapter set when the attempted transition is represented in the type system
- calling plugin or adapter client namespaces that are not present on the inferred `@commerce-kit/client` result

These guarantees are bounded by TypeScript's static analysis model. They apply to inferred config, known namespaces, and typed transition inputs; they do not claim to reject every invalid runtime-computed string or dynamically assembled access path.

## Plugin-gated and adapter-gated surface area

Type removal is part of the contract, not an implementation detail.

- Without `@commerce-kit/marketplace`, vendor and `vendorOrder` APIs do not exist at compile time.
- Marketplace-installed route and client namespaces exist only when the marketplace plugin contributes them through the documented plugin contract.
- With no `fulfillment` configuration, fulfillment APIs do not exist at compile time.

This mirrors the runtime architecture: inactive capabilities do not expose dormant surface area.

## State inference

`OrderStatus` is derived from the base state machine plus installed plugin state extensions and any states contributed by activated optional adapters. A plugin or adapter that is not installed cannot contribute states or transitions to the inferred type surface.

For marketplace specifically:

- core `OrderStatus` keeps the simple-store base lifecycle
- `@commerce-kit/marketplace` may add declared states and transitions only where the customer-facing core order lifecycle must expose marketplace-aware progression
- `vendorOrder.status` is a separate marketplace-owned lifecycle and must not be treated as the same type as core `OrderStatus` unless the marketplace package explicitly defines and names such a projection
- any marketplace-managed `vendorOrder.status` lifecycle must be validated by the marketplace package at runtime; it is not automatically part of the core `OrderStatus` union

## Client inference

`createCommerceClient<typeof commerce>()` from `@commerce-kit/client` mirrors the server's installed plugin and adapter surface for typed client usage. The client SDK is inferred from the configured server instance rather than from a separate manually declared client-shape contract.

The canonical mental model for both the in-process server runtime and the HTTP-backed client runtime is:

```ts
commerce.domain.method(...)
```

Illustrative shape:

```ts
commerce.products.list(...)
commerce.products.get(...)
commerce.products.create(...)
commerce.products.update(...)
commerce.products.archive(...)

commerce.orders.list(...)
commerce.orders.get(...)
commerce.orders.checkout(...)
commerce.orders.cancel(...)
commerce.orders.setPaid(...)
commerce.orders.refund(...)
```

This contract is capability-based rather than assuming uniform CRUD across every namespace.

- mutable domains expose only the CRUD-like methods they actually support
- immutable or workflow-driven domains omit unsupported mutating methods entirely at compile time
- workflow operations such as `orders.checkout(...)`, `orders.cancel(...)`, and `orders.refund(...)` remain explicit domain methods rather than being forced into generic CRUD naming

Method omission is part of the type contract, not a runtime fallback strategy. If a namespace or operation is not supported by the configured core domain, installed plugin set, or activated adapter set, it must not exist on the inferred API surface.

### Object-only method inputs

Every public SDK method accepts a single object argument. This keeps signatures stable as APIs add IDs, version checks, idempotency, filtering, or workflow-specific options.

Normative conventions:

- `create(...)` uses direct resource fields.
- `update(...)` uses `{ id, data }`.
- nested updates use explicit parent/child IDs plus `data`.
- workflow actions use action-specific fields directly.
- query methods use one structured query object.

Examples:

```ts
await commerce.products.get({ id: "prod_123" })

await commerce.products.create({
  title: "Linen Shirt",
  slug: "linen-shirt",
})

await commerce.products.update({
  id: "prod_123",
  data: { title: "Updated Linen Shirt" },
})

await commerce.products.variants.update({
  productId: "prod_123",
  variantId: "var_123",
  data: { price: { amount: 14999, currency: "USD" } },
})
```

### Inferred query types

Collection-reading methods should infer domain-specific query shapes rather than exposing loose untyped records.

The approved v1 query model is Drizzle-inspired but plain-data and transport-safe:

```ts
await commerce.products.list({
  where: {
    status: "active",
    price: { gte: 1000, lte: 5000 },
  },
  fields: ["id", "title", "slug", "price"],
  expand: ["variants", "media"],
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
  cursor: "next_cursor",
})
```

Normative expectations:

- `list` is the standard collection reader and supports pagination, sorting, and structured filtering
- `search` exists only for namespaces that need semantics beyond structured filtering
- `where` and `orderBy` are inferred per namespace rather than shared as one loose global shape
- `fields` and `expand` are inferred per namespace for read methods that support projection and relation expansion
- callback-based query builders and raw SQL-like public query expressions are out of scope for the client API

Illustrative generic forms:

```ts
type ListInput<TWhere, TOrderBy> = {
  where?: TWhere
  orderBy?: TOrderBy[]
  fields?: string[]
  expand?: string[]
  limit?: number
  cursor?: string
}

type SearchInput<TWhere, TOrderBy> = {
  query: string
  where?: TWhere
  orderBy?: TOrderBy[]
  limit?: number
  cursor?: string
}
```

## Core domains

The following domains are built into core and available without additional plugins or adapters:

| Domain | Methods | Description |
|--------|---------|-------------|
| `products` | `list`, `get`, `create`, `update`, `archive`, `versions`, `getVersion` | Product catalog with version history |
| `orders` | `list`, `get`, `checkout`, `calculate`, `cancel`, `confirm`, `setPaid`, `refund` | Order checkout, calculation, and lifecycle workflows |
| `payments` | `list`, `get` | Immutable payment ledger |
| `customers` | `list`, `get`, `create`, `update` | Customer records |
| `categories` | `list`, `get`, `create`, `update`, `archive` | Product categories |
| `cart` | `get`, `add`, `update`, `remove`, `clear`, `calculate` | Client-side cart with auto-recalculation; optional server persistence — see [22-cart.md](./22-cart.md) |

### Product versioning

Products maintain a full version history to preserve historical pricing and details at the time of each order. Each version records:

- `title`, `slug`, `description`
- `price`, `compareAtPrice`
- `categoryId`, `images`, `variants`
- `changedAt`, `changedBy`

The current product includes `version: number` and `versions: ProductVersion[]` for audit and historical queries.

### Adapter-gated domains

The following domain is only available when its corresponding adapters are configured:

- `fulfillment` — configured fulfillment methods, rates/prices, labels, tracking, local delivery, pickup, and digital delivery

## Requirements

- TypeScript 5.0+
- TypeScript-only support
- Features depend on `const` type parameters, `satisfies`, and variadic tuples

## Future RFCs

- Additional inferred helper types beyond `InferCommerceTypes`
- Stronger entity-ID typing guarantees if identity contracts are formalized elsewhere in the architecture set
