# Client API Design

> Superseded note: the canonical client API direction has moved into `docs/architecture/60-type-inference.md`, `docs/architecture/80-framework-adapters.md`, and `docs/examples/getting-started.md`. Those docs now use the domain-first, object-only API with `orders.checkout(...)`, unified fulfillment, and high-level order payment workflows.

**Date:** 2026-04-26
**Status:** Approved

## Goal

Define the developer-facing Commerce Kit client API so both in-process server usage and HTTP-backed client usage share the same domain-first shape, capability model, and inferred types.

## Decisions

### 1. One shared API shape across runtimes

- Commerce Kit should support both in-process server-side usage and HTTP-backed client-side usage.
- Both runtimes should expose the same namespace-first mental model: `commerce.domain.method(...)`.
- The server runtime should be represented by a top-level `CommerceKit` instance.
- The HTTP runtime should be represented by `createCommerceClient(...)` from `@commerce-kit/client`.
- The public API shape should be derived from one shared domain contract rather than maintained separately for server and client runtimes.

### 2. Domain-first service namespaces

Each domain exposes a small, intentional service object rather than a generic bag of endpoints.

Illustrative shape:

```ts
const commerce = new CommerceKit({ ... })

await commerce.products.list(...)
await commerce.products.get(id)
await commerce.products.create(...)
await commerce.products.update(id, ...)
await commerce.products.archive(id)

await commerce.orders.list(...)
await commerce.orders.get(id)
await commerce.orders.create(...)
await commerce.orders.transition(id, ...)
```

- Mutable domains should expose CRUD-like methods where appropriate.
- Immutable or workflow-driven domains should omit unsupported mutating methods entirely.
- Domain-specific operations such as `orders.transition(...)` should remain explicit instead of being forced into fake CRUD.

### 3. Capability omission is part of the API contract

- Unsupported methods must be omitted at compile time, not exposed and rejected later at runtime.
- The same omission rules should apply to both server and HTTP clients.
- Plugin and optional adapter namespaces should follow the same rule: they exist only when installed or configured.
- The API contract is therefore capability-driven rather than assuming full CRUD for every namespace.

Examples:

- `products`: `list`, `get`, `create`, `update`, `archive`, optionally `search`
- `orders`: `list`, `get`, `create`, `transition`
- read-only namespaces: `list`, `get`, optionally `search`

### 4. Drizzle-inspired query shape

Collection reads should use a Drizzle-inspired plain-object query shape that remains serializable across HTTP.

Illustrative `list` input:

```ts
await commerce.products.list({
  where: {
    status: "active",
    price: { gte: 1000, lte: 5000 },
    categoryId: { in: ["cat_1", "cat_2"] },
  },
  orderBy: [
    { createdAt: "desc" },
    { title: "asc" },
  ],
  limit: 20,
  cursor: "next_cursor",
})
```

Illustrative `search` input:

```ts
await commerce.products.search({
  query: "linen shirt",
  where: {
    status: "active",
    inventory: { gt: 0 },
  },
  orderBy: [{ relevance: "desc" }],
  limit: 12,
})
```

Rules:

- `list` should be the default collection reader and always support pagination, sorting, and structured filtering.
- `search` should exist only for domains that need semantics beyond structured filtering.
- `where` and `orderBy` should be domain-specific and inferred rather than loose generic records.
- The public API should not expose callback-based query builders or raw SQL-like expressions.

Recommended generic shapes:

```ts
type ListInput<TWhere, TOrderBy> = {
  where?: TWhere
  orderBy?: TOrderBy[]
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

type PageResult<T> = {
  items: T[]
  pageInfo: {
    nextCursor: string | null
    prevCursor?: string | null
    hasMore: boolean
  }
}
```

### 5. Shared contract with two runtime implementations

The implementation should be split into three layers:

1. a domain contract layer that defines namespaces, methods, inputs, outputs, and capability flags
2. a server runtime that binds those methods to in-process handlers and core services
3. an HTTP runtime that binds those same methods to fetch calls and canonical route/action endpoints

This keeps namespace names, method names, capability omission, and type inference in one place.

### 6. Transport mapping

Where canonical REST routes already exist, the HTTP client should map directly to them:

- `products.list` -> `GET /products`
- `products.get` -> `GET /products/:id`
- `products.create` -> `POST /products`
- `products.update` -> `PATCH /products/:id`
- `products.archive` -> `DELETE /products/:id`

Workflow actions should use explicit action routes:

- `orders.transition` -> `POST /orders/:id/transition`

`search` should only get a dedicated route if its semantics are materially distinct from `list`. If not, `list` should remain the standard collection reader.

### 7. Type inference model

The approved API must align with the existing architecture decision that `createCommerce(...)` is the source of truth for types.

Illustrative flow:

```ts
const commerce = createCommerce({ ... })

type Types = InferCommerceTypes<typeof commerce>
type ClientApi = Types["ClientApi"]
```

Inference rules:

- core namespaces exist only when core enables them
- plugin namespaces exist only when plugins are installed
- adapter-gated namespaces exist only when matching adapters are configured
- methods are omitted based on domain capabilities
- domain-specific `where` and `orderBy` types are part of the inferred namespace contract

### 8. Error model

- The server runtime should throw structured Commerce Kit errors.
- The HTTP runtime should normalize transport failures into a consistent client error model.
- Validation, not found, unauthorized, forbidden, and domain-state failures should remain easy to distinguish across both runtimes.

### 9. Testing priorities

The eventual implementation should be verified at three layers:

- type tests for namespace and method omission
- server runtime tests for in-process namespace dispatch
- HTTP client tests for route mapping and query serialization

Most important cases:

- `products.update` exists while `orders.update` does not
- `shipping.*` and `delivery.*` are absent when adapters are not configured
- plugin namespaces appear only when installed
- `list` and `search` serialize `where` and `orderBy` consistently across runtimes
