# Framework Adapters

## Purpose

Define how Commerce Kit is exposed over HTTP, how framework adapters map requests into core route handlers, how request context is resolved, how the client SDK is inferred, and how webhook and error handling work across frameworks.

## Non-goals

- Monorepo layout, package release policy, and build tooling ownership
- CLI config discovery and migration/generation workflow
- Core commerce rules that are not specific to HTTP exposure

## Core decisions

### HTTP is an adapter layer, not the core

Commerce Kit separates API behavior from transport. Core logic is pure TypeScript with no HTTP or framework dependency. Framework packages only map HTTP requests into canonical core handlers.

The same core logic function is used across all framework adapters. HTTP adapters must not duplicate domain logic.

The canonical framework package inventory lives in [70-packages-and-monorepo.md](./70-packages-and-monorepo.md). `@commerce-kit/orpc` is post-v1 and is not part of the v1 HTTP contract.

## Canonical route handler contract

Core owns one framework-agnostic route layer:

```ts
interface RouteRequest {
  params: Record<string, string>
  query: Record<string, string>
  body: unknown
  context: RequestContext
}

interface RouteResponse {
  status: number
  body: unknown
  headers?: Record<string, string>
}
```

Canonical mounted route set in core includes:

```text
GET    /products
GET    /products/:id
POST   /products
PATCH  /products/:id
DELETE /products/:id   (maps to product archival/status transition, not hard deletion)
GET    /products/:id/variants
POST   /products/:id/variants
PATCH  /products/:id/variants/:variantId

GET    /orders
GET    /orders/:id
POST   /orders/checkout
POST   /orders/:id/cancel
POST   /orders/:id/confirm
POST   /orders/:id/set-paid
POST   /orders/:id/refund

GET    /fulfillment/methods        (only when fulfillment is configured)
POST   /fulfillment/methods        (only when fulfillment is configured)
PATCH  /fulfillment/methods/:id    (only when fulfillment is configured)
```

Required payment behavior is exposed through core order operations and server-side APIs, not through a separate mandatory public `/payments` REST surface in v1.

- customer checkout and payment initiation happen through `orders.checkout(...)`
- order-level workflows such as `orders.cancel(...)`, `orders.setPaid(...)`, and `orders.refund(...)` coordinate payment adapters under the hood
- direct payment lifecycle operations such as authorize/capture/cancel are adapter or trusted server internals by default
- payment ledger reads may be exposed through `/payments`, but a standalone public payment mutation route family is not required by this architecture doc
- regardless of HTTP shape, the payment adapter contract remains the canonical execution boundary for `authorize`, `capture`, `refund`, and `cancel`

Additional HTTP routes may be mounted when installed plugins expose adapter-visible API extensions. The mapping from plugin API surface to HTTP routes is owned by core and the framework adapters, not by arbitrary plugin-side router registration.

For route contribution, the plugin contract is namespace- and handler-based rather than raw router injection: plugins contribute typed API surfaces and declarative route definitions, and core/framework adapters install those definitions into the HTTP layer.

Authorization ownership is layered:

- applications own actor identity resolution through `resolveContext`
- core owns generic request-context plumbing plus any invariant checks required by built-in operations
- applications own policy decisions such as which authenticated actor roles may call mutating core routes
- plugins own any additional domain-specific authorization rules for the plugin-owned mutating operations they introduce

Framework adapters enforce the result of core/plugin authorization decisions in HTTP responses, but they do not define business authorization policy themselves.

Collision policy is fail-fast:

- plugin API namespaces must be unique across the installed plugin set
- plugin-contributed HTTP routes must not conflict with core routes or with routes from other installed plugins
- any namespace or route collision is a configuration error during Commerce Kit initialization, not a last-write-wins behavior

Marketplace-owned vendor routes are not part of the base route set. They are installed only when `@commerce-kit/marketplace` is present.

Example marketplace-installed routes:

```text
GET    /vendors
GET    /vendors/:id
POST   /vendors
PATCH  /vendors/:id
POST   /vendors/:id/suspend
GET    /vendors/:id/orders
POST   /vendor-orders/:id/transition
```

## Input validation

Commerce Kit validates request input internally. Transport-shape validation and parsing may use shared schemas such as Zod, and invalid input returns HTTP 400 with a structured validation payload.

Validation ownership is layered:

- core and installed plugins own domain validation and invariants
- the canonical route layer owns request-shape validation for route inputs
- framework adapters only translate HTTP requests/responses into that validation pipeline; they do not own domain validation rules themselves

## resolveContext

`resolveContext` is the authentication and request-context seam. Commerce Kit does not read sessions or tokens directly.

```ts
interface ResolveContextFn<TRequest> {
  (req: TRequest): Promise<{
    customerId: string | null
    actorId?: string | null
    actorType?: string
    roles?: string[]
    permissions?: string[]
    locale?: string
    metadata?: Record<string, unknown>
  }>
}
```

`@commerce-kit/better-auth` provides a prebuilt resolver integration, but Commerce Kit itself remains auth-agnostic.

The base request-context contract is intentionally actor-capable but domain-neutral: applications may supply actor, role, or permission data needed for policy enforcement without introducing vendor or marketplace concepts into core by default.

If an installed plugin such as `@commerce-kit/marketplace` needs additional domain-specific context, that context remains plugin-gated and extension-specific rather than part of the base `ResolveContextFn` contract.

## Framework mappings

### Hono

- Mount API routes via `honoAdapter(commerce, { resolveContext })`
- Mount webhook handling separately via `handleWebhook`

### Elysia

- Mount API routes via `elysiaPlugin(commerce, { prefix, resolveContext })`
- Webhook route must opt out of body parsing with `parse: "none"`

### Express

- Mount API routes via `expressRouter(commerce, { resolveContext })`
- Webhook route must use `express.raw({ type: "*/*" })` before the handler

### Fastify

- Register API routes through the Fastify plugin with `commerce` and `resolveContext`
- Preserve webhook raw bodies with `addContentTypeParser("*", { parseAs: "buffer" }, ...)`

### Next.js App Router

- Use a shared handler for HTTP verbs on catch-all API routes
- Use a dedicated webhook handler for webhook routes

## Client SDK

`@commerce-kit/client` is the canonical fetch-based SDK for v1 and owns the `createCommerceClient` runtime factory.

- No extra client runtime dependency beyond fetch
- Client types are inferred from `typeof commerce`
- `commerce-kit` owns the server-side types that the client package consumes for inference; the client runtime itself is not bundled into core
- Plugin client methods exist only when the matching server-side plugin is installed
- Fulfillment client namespaces follow the same compile-time activation rules as the server

The server-side runtime and the fetch-based client runtime should expose the same public namespace-first shape:

```ts
commerce.domain.method(...)
```

Illustrative core examples:

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

This is a shared contract, not two separately designed APIs. The server runtime may execute methods in-process while `@commerce-kit/client` executes them over HTTP, but both must preserve the same namespace and method vocabulary.

Unsupported methods are omitted rather than exposed as always-present endpoints that later fail. For example, an immutable or workflow-driven namespace may expose `list` and `get` without also exposing `update` or `delete`, and optional namespaces such as `fulfillment` exist only when configured.

### Client method to HTTP mapping

Where canonical REST routes exist, the fetch client maps directly to them:

- `products.list` -> `GET /products`
- `products.get` -> `GET /products/:id`
- `products.create` -> `POST /products`
- `products.update` -> `PATCH /products/:id`
- `products.archive` -> `DELETE /products/:id`
- `orders.list` -> `GET /orders`
- `orders.get` -> `GET /orders/:id`
- `orders.checkout` -> `POST /orders/checkout`

Workflow methods use explicit action routes instead of pretending to be CRUD:

- `orders.cancel` -> `POST /orders/:id/cancel`
- `orders.setPaid` -> `POST /orders/:id/set-paid`
- `orders.refund` -> `POST /orders/:id/refund`

The same route-mapping principle applies to plugin-owned namespaces and methods when installed, subject to the collision rules above.

### Query transport shape

Collection reads use a Drizzle-inspired plain-object query shape that remains serializable across HTTP.

Illustrative shape:

```ts
await commerce.products.list({
  where: {
    status: "active",
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

Normative expectations:

- `list` is the standard collection reader and supports pagination, sorting, and structured filtering
- `search` exists only for namespaces that need semantics beyond structured filtering
- if `search` exists, it follows the same transport-safe query model plus a search term such as `query`
- framework adapters transport query inputs; they do not redefine domain query semantics

### Core domains

The core domains built into `commerce-kit` are:

| Domain | Methods | HTTP Routes |
|--------|---------|-------------|
| `products` | `list`, `get`, `create`, `update`, `archive`, `versions`, `getVersion` | `/products` + versions |
| `orders` | `list`, `get`, `checkout`, `cancel`, `confirm`, `setPaid`, `refund` | `/orders` + checkout + workflow actions |
| `payments` | `list`, `get` | `/payments` |
| `customers` | `list`, `get`, `create`, `update` | `/customers` |
| `categories` | `list`, `get`, `create`, `update`, `archive` | `/categories` |
| `cart` | `get`, `add`, `update`, `remove`, `clear` | client-side primitive; optional `/cart` when server cart persistence is configured |
| `fulfillment` | `methods.list`, `methods.get`, `methods.create`, `methods.update`, `methods.archive`, `track`, `createLabel` | `/fulfillment` when configured |

Products maintain a full version history. Each update creates a new version preserving the previous state for audit and historical pricing.

## Webhooks

Webhook processing bypasses the normal REST route handlers because signature verification requires raw body bytes.

### Processing flow

1. Read raw request body without prior JSON parsing
2. Resolve the adapter by `type` and `adapterId`
3. Call `adapter.verifyWebhook(...)`
4. Reject invalid signatures with HTTP 401
5. Parse a verified `WebhookEvent`
6. Apply idempotency checks
7. Perform the core event handling for the event type
8. Fire plugin webhook handlers for that verified event
9. Return HTTP 200 only after successful verified processing and plugin dispatch

### Webhook idempotency

Webhook idempotency is enforced at the database boundary using provider-specific unique identifiers owned by the relevant adapter surface. For payment events, that boundary is the provider event identifier persisted separately from the payment transaction reference. Other webhook surfaces must define an equivalent adapter-owned event key before claiming exactly-once processing.

### Webhook URL convention

```text
/webhooks/payment/:adapterId
/webhooks/fulfillment/:adapterId
/webhooks/payout/:adapterId
```

### Raw body preservation by framework

| Framework | Raw body strategy |
|---|---|
| Hono | `c.req.arrayBuffer()` |
| Elysia | `parse: "none"` |
| Express | `express.raw({ type: "*/*" })` |
| Fastify | custom buffer content-type parser |
| Next.js | `req.arrayBuffer()` |

Plugins receive verified webhook events, never raw unverified HTTP payloads. Plugin webhook handlers are downstream consumers of adapter-verified events, not alternate verification hooks.

## Error to HTTP mapping

This mapping is defined once in core and reused by every framework adapter.

| Error | Status |
|---|---|
| `CommerceConfigError` | 500 |
| `CommercePluginError` | 500 |
| `CommerceOrderError` | 422 |
| `CommerceStateError` | 422 |
| `CommercePaymentError` | 402 |
| `MoneyCurrencyError` | 422 |
| Validation error | 400 |
| Not found | 404 |
| Unauthorized | 401 |
| Forbidden | 403 |
| Invalid webhook signature | 401 |

## Future RFCs

- oRPC and OpenAPI generation after v1
- Any new framework adapter package
- Any change to webhook transport conventions or route ownership
