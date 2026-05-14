# Server SDK

## Purpose

Define the full `commerce.*` server SDK: namespace structure, method signatures, inputs, return types, context binding, and shared conventions.

HTTP route mapping lives in [80-framework-adapters.md](./80-framework-adapters.md). Error behavior lives in [15-errors.md](./15-errors.md). Type inference lives in [60-type-inference.md](./60-type-inference.md).

## Non-goals

- HTTP transport and route mounting
- Client SDK (`@commerce-kit/client`) method shapes — those mirror this API but run over HTTP
- Authorization policy — the SDK is privileged server code; apps own policy

## Core decisions

### The server SDK is always privileged

Direct `commerce.*` calls are trusted server-side code. There is no implicit customer scoping. If you want only a customer's orders, pass `where: { customerId }` explicitly. The HTTP layer applies `resolveContext`-based scoping automatically; the server SDK does not.

### Context binding with `withContext`

By default, operations carry no actor context. Hooks receive no `actorId` or `roles`. This is correct for background jobs and cron tasks.

When actor context matters — route handlers, admin operations, audit logs — bind it once:

```ts
const c = commerce.withContext({
  customerId: session.customerId,
  actorId: session.userId,
  roles: session.roles,
  merchantId: session.merchantId,    // when tenancy.merchants is enabled
  branchId: session.branchId,    // when tenancy.branches is enabled
})

// All calls on `c` pass this context to hooks and tenancy-aware tables
await c.orders.cancel({ id: 'ord_123' })
await c.products.archive({ id: 'prod_456' })
```

`withContext` returns a new SDK instance with the same shape. It does not mutate `commerce`.

#### Recipe: binding context in a route handler

`withContext` is the seam for every tenancy-aware operation. In a route handler, resolve the request's actor and tenancy once, then call the scoped instance for the rest of the request:

```ts
// app/api/storefront/orders/route.ts (Next.js)
export async function GET(req: Request) {
  const session = await resolveSession(req)
  const store = commerce.withContext({
    actorId: session.userId,
    customerId: session.customerId,
    merchantId: session.merchantId,   // present when tenancy.merchants is on
    branchId: session.branchId,       // present when tenancy.branches is on
  })

  const orders = await store.orders.list()       // ✅ scoped to this merchant
  return Response.json(orders)
}
```

Bind context once per request. Do not call `withContext` inside loops or hooks — the returned SDK is meant to be reused for the lifetime of the request. For role-aware admin/storefront routing, see [12-tenancy.md → Admin UI patterns](./12-tenancy.md#admin-ui-patterns).

```ts
interface RequestContext {
  customerId?: string | null
  actorId?: string | null
  actorType?: string
  roles?: string[]
  permissions?: string[]
  merchantId?: string                // present when tenancy.merchants is on
  branchId?: string                // present when tenancy.branches is on
  locale?: string
  metadata?: Record<string, unknown>
}
```

When tenancy is active, `merchantId` and `branchId` on the context drive automatic write inference and hierarchical reads for any table declaring `merchant()` / `branch()` columns. See [12-tenancy.md](./12-tenancy.md).

### Shared types

```ts
type Money = {
  amount: number    // integer minor units (e.g. 1999 = $19.99)
  currency: string  // ISO 4217 (e.g. 'USD', 'SAR')
}

type ListResult<T> = {
  items: T[]
  nextCursor?: string
}
```

### Shared list input

All `list` methods accept a common query shape. `where` and `orderBy` are inferred per namespace.

```ts
type ListInput<TWhere, TOrderBy, TExpand> = {
  where?: TWhere
  orderBy?: TOrderBy[]
  fields?: string[]
  expand?: TExpand[]
  limit?: number
  cursor?: string
}
```

---

## `products`

### `products.list`

```ts
commerce.products.list({
  where?: {
    status?: 'active' | 'archived'
    categoryId?: string | { in: string[] }
    price?: { gte?: number; lte?: number }
  }
  orderBy?: { createdAt?: 'asc' | 'desc'; title?: 'asc' | 'desc'; price?: 'asc' | 'desc' }[]
  fields?: (keyof Product)[]
  expand?: ('variants' | 'category' | 'media')[]
  limit?: number
  cursor?: string
}): Promise<ListResult<Product>>
```

### `products.get`

```ts
commerce.products.get({
  id: string
  expand?: ('variants' | 'category' | 'media')[]
}): Promise<Product>
```

### `products.create`

```ts
commerce.products.create({
  title: string
  slug: string
  description?: string
  categoryId?: string
  status?: 'active' | 'archived'
  metadata?: Record<string, unknown>
}): Promise<Product>
```

### `products.update`

```ts
commerce.products.update({
  id: string
  data: {
    title?: string
    slug?: string
    description?: string
    categoryId?: string
    status?: 'active' | 'archived'
    metadata?: Record<string, unknown>
  }
}): Promise<Product>
```

Each update creates a new version. The previous state is preserved in version history.

### `products.archive`

```ts
commerce.products.archive({ id: string }): Promise<Product>
```

### `products.versions`

```ts
commerce.products.versions({
  productId: string
  limit?: number
  cursor?: string
}): Promise<ListResult<ProductVersion>>
```

### `products.getVersion`

```ts
commerce.products.getVersion({
  productId: string
  version: number
}): Promise<ProductVersion>
```

---

## `products.variants`

Variants are a nested namespace under `products`.

### `products.variants.list`

```ts
commerce.products.variants.list({
  productId: string
  where?: { status?: 'active' | 'archived' }
}): Promise<ListResult<ProductVariant>>
```

### `products.variants.get`

```ts
commerce.products.variants.get({
  productId: string
  variantId: string
}): Promise<ProductVariant>
```

### `products.variants.create`

```ts
commerce.products.variants.create({
  productId: string
  sku: string
  price: Money
  attributes: Record<string, string>
  status?: 'active' | 'archived'
  metadata?: Record<string, unknown>
}): Promise<ProductVariant>
```

### `products.variants.update`

```ts
commerce.products.variants.update({
  productId: string
  variantId: string
  data: {
    sku?: string
    price?: Money
    attributes?: Record<string, string>
    status?: 'active' | 'archived'
    metadata?: Record<string, unknown>
  }
}): Promise<ProductVariant>
```

### `products.variants.archive`

```ts
commerce.products.variants.archive({
  productId: string
  variantId: string
}): Promise<ProductVariant>
```

---

## `orders`

`Order` is a discriminated union over `status`. The methods reachable on `commerce.orders` for a narrowed order match the state machine in [20-data-model.md → Order state machine](./20-data-model.md#order-state-machine) — invalid transitions are removed from the type, not just rejected at runtime.

```ts
const order = await commerce.orders.get({ id })
//   ^? Order<'placed'> | Order<'confirmed'> | Order<'completed'> | …

if (order.status === 'placed') {
  await commerce.orders.cancel(order)    // ✅ allowed from 'placed'
  await commerce.orders.confirm(order)   // ✅ allowed from 'placed'
}

if (order.status === 'completed') {
  await commerce.orders.cancel(order)    // ❌ Property 'cancel' does not exist on Order<'completed'>
  await commerce.orders.refund(order)    // ✅ allowed from 'completed'
}
```

Plugin-registered states flow into the same union — a plugin that adds `fraud_review_pending` extends `OrderStatus` and the per-status method maps accordingly. See [40-plugin-system.md → Order state extensions](./40-plugin-system.md).

### `orders.list`

```ts
commerce.orders.list({
  where?: {
    customerId?: string
    status?: OrderStatus | OrderStatus[]
    paymentAdapterId?: string
    createdAt?: { gte?: Date; lte?: Date }
  }
  orderBy?: { createdAt?: 'asc' | 'desc' }[]
  expand?: ('items' | 'customer' | 'payments')[]
  limit?: number
  cursor?: string
}): Promise<ListResult<Order>>
```

### `orders.get`

```ts
commerce.orders.get({
  id: string
  expand?: ('items' | 'customer' | 'payments')[]
}): Promise<Order>
```

### `orders.checkout`

Creates an order and initiates payment. The payment adapter executes `authorize` internally.

```ts
commerce.orders.checkout({
  items: { variantId: string; quantity: number }[]
  customerId?: string            // null for guest checkout
  fulfillmentMethodId?: string   // required when fulfillment is configured
  fulfillmentTypeData?: unknown  // typed payload validated against the method's type schema
  payment: {
    adapterId: string            // typed as union of registered payment adapter keys
    metadata?: Record<string, unknown>  // adapter-specific fields
  }
  couponCode?: string
  metadata?: Record<string, unknown>
}): Promise<{
  order: Order
  paymentReference: string
  flow: 'synchronous' | 'redirect' | 'inline'
  paymentUrl?: string            // present when flow === 'redirect'
  inlineSecret?: string          // present when flow === 'inline'
}>
```

The `flow` field comes from the selected adapter's `capabilities.flow`. The caller dispatches on it:

| `flow` | What the caller does | Order state after checkout |
|---|---|---|
| `'synchronous'` | Display the order as awaiting payment collection (COD, manual). Call `orders.setPaid` when payment is collected. | `placed`, payment `initiated` |
| `'redirect'` | Redirect the customer to `paymentUrl`. | `placed`, payment `initiated`; webhook confirms |
| `'inline'` | Pass `inlineSecret` to the provider's JS SDK to complete payment in-page. | `placed`, payment `initiated`; webhook confirms |

### `orders.calculate`

Pure read — computes totals for a prospective order without creating anything. Uses the same pricing pipeline as checkout.

```ts
commerce.orders.calculate({
  items: { variantId: string; quantity: number }[]
  fulfillmentMethodId?: string
  couponCode?: string
  customerId?: string
}): Promise<{
  subtotal: Money
  discountTotal: Money
  taxTotal: Money
  fulfillmentTotal: Money
  total: Money
  items: {
    variantId: string
    quantity: number
    unitPrice: Money
    total: Money
  }[]
}>
```

### `orders.confirm`

Merchant or system accepts the order for processing. Transitions `placed → confirmed`.

```ts
commerce.orders.confirm({ id: string }): Promise<Order>
```

### `orders.cancel`

```ts
commerce.orders.cancel({
  id: string
  reason?: string
}): Promise<Order>
```

Valid from `draft`, `placed`, `confirmed`, and `processing`. Triggers payment cancellation via the payment adapter when applicable.

### `orders.setPaid`

Marks the order as paid. Valid only for payments whose adapter declares `capabilities.flow === 'synchronous'` — COD, manual bank transfer, in-store payment. Calling `setPaid` on a `redirect` or `inline` payment throws `CommerceStateError`; those flows must be confirmed via webhook.

```ts
commerce.orders.setPaid({
  id: string
  paymentReference?: string
}): Promise<Order>
```

### `orders.refund`

```ts
commerce.orders.refund({
  id: string
  amount?: Money            // partial refund when provided; full refund when omitted
  reason?: string
  mode?: 'auto' | 'refund-only' | 'void-only'
}): Promise<Order>
```

`orders.refund` is the only order-level payment reversal operation. There is no `orders.void` — void is a payment-adapter mechanic, not an order concern. The operation orchestrates void and refund per payment automatically.

Behavior per eligible payment (in `mode: 'auto'`, the default):

1. If the payment is in a voidable state (typically `authorized`) AND the adapter declares `capabilities.supportsVoid: true`, call `adapter.cancel(providerReference)` first.
2. If void fails or the payment is not in a voidable state, call `adapter.refund(providerReference, amount, reason)`.
3. The payment ledger receives a new append-only row recording either the `voided` or `refunded` outcome.

`mode: 'refund-only'` skips the void attempt and always uses `refund`. `mode: 'void-only'` requires the payment to be in a voidable state and the adapter to support void; otherwise the operation errors. Default is `'auto'`.

The void-first orchestration matches the common provider pattern where voiding an authorization is cheaper than refunding a capture. Adapters that do not differentiate (or that auto-capture) declare `supportsVoid: false` and only refund.

---

## `payments`

The payment ledger is append-only and immutable. No mutation methods are exposed.

### `payments.list`

```ts
commerce.payments.list({
  where?: {
    orderId?: string
    adapterId?: string
    type?: string
    status?: string
  }
  orderBy?: { createdAt?: 'asc' | 'desc' }[]
  limit?: number
  cursor?: string
}): Promise<ListResult<Payment>>
```

### `payments.get`

```ts
commerce.payments.get({ id: string }): Promise<Payment>
```

---

## `customers`

### `customers.list`

```ts
commerce.customers.list({
  where?: {
    email?: string
    createdAt?: { gte?: Date; lte?: Date }
  }
  orderBy?: { createdAt?: 'asc' | 'desc'; name?: 'asc' | 'desc' }[]
  limit?: number
  cursor?: string
}): Promise<ListResult<Customer>>
```

### `customers.get`

```ts
commerce.customers.get({ id: string }): Promise<Customer>
```

### `customers.create`

```ts
commerce.customers.create({
  email: string
  name?: string
  phone?: string
  metadata?: Record<string, unknown>
}): Promise<Customer>
```

### `customers.update`

```ts
commerce.customers.update({
  id: string
  data: {
    email?: string
    name?: string
    phone?: string
    metadata?: Record<string, unknown>
  }
}): Promise<Customer>
```

---

## `categories`

### `categories.list`

```ts
commerce.categories.list({
  where?: {
    parentId?: string | null   // null = root categories only
    status?: 'active' | 'archived'
  }
  orderBy?: { name?: 'asc' | 'desc'; createdAt?: 'asc' | 'desc' }[]
  limit?: number
  cursor?: string
}): Promise<ListResult<Category>>
```

### `categories.get`

```ts
commerce.categories.get({ id: string }): Promise<Category>
```

### `categories.create`

```ts
commerce.categories.create({
  name: string
  slug: string
  parentId?: string
  metadata?: Record<string, unknown>
}): Promise<Category>
```

### `categories.update`

```ts
commerce.categories.update({
  id: string
  data: {
    name?: string
    slug?: string
    parentId?: string
    metadata?: Record<string, unknown>
  }
}): Promise<Category>
```

### `categories.archive`

```ts
commerce.categories.archive({ id: string }): Promise<Category>
```

---

## `cart`

`cart` methods are available on the server SDK only when server cart persistence is configured. When not configured, `commerce.cart` does not exist at compile time. See [22-cart.md](./22-cart.md).

For calculating cart totals without persistence, use `orders.calculate`.

### `cart.get`

```ts
commerce.cart.get({ id: string }): Promise<Cart>
```

### `cart.add`

```ts
commerce.cart.add({
  id: string
  variantId: string
  quantity: number
}): Promise<Cart>
```

### `cart.update`

```ts
commerce.cart.update({
  id: string
  itemId: string
  quantity: number
}): Promise<Cart>
```

### `cart.remove`

```ts
commerce.cart.remove({
  id: string
  itemId: string
}): Promise<Cart>
```

### `cart.clear`

```ts
commerce.cart.clear({ id: string }): Promise<Cart>
```

---

## `merchants`

`merchants` exists only when `tenancy.merchants: true`. See [12-tenancy.md](./12-tenancy.md).

```ts
commerce.merchants.list({ where?, orderBy?, limit?, cursor? }): Promise<ListResult<Merchant>>
commerce.merchants.get({ id: string }): Promise<Merchant>
commerce.merchants.create({ name, slug, metadata? }): Promise<Merchant>
commerce.merchants.update({
  id: string
  data: {
    name?: string
    slug?: string
    status?: 'active' | 'suspended' | 'archived'
    autoDispatchDelivery?: boolean | null      // null clears the override; falls back to app default
    defaultDeliveryMethodId?: string | null
    metadata?: Record<string, unknown>
  }
}): Promise<Merchant>
commerce.merchants.archive({ id: string }): Promise<Merchant>
```

---

## `branches`

`branches` exists only when `tenancy.branches: true`. See [12-tenancy.md](./12-tenancy.md).

```ts
commerce.branches.list({ where?: { merchantId? }, ... }): Promise<ListResult<Branch>>
commerce.branches.get({ id: string }): Promise<Branch>
commerce.branches.create({
  merchantId: string
  name: string
  slug: string
  address: Address
  coordinates?: Coordinates
  timezone: string
  metadata?: Record<string, unknown>
}): Promise<Branch>
commerce.branches.update({
  id: string
  data: {
    name?: string
    slug?: string
    status?: 'active' | 'archived'
    address?: Address
    coordinates?: Coordinates | null
    timezone?: string
    defaultDeliveryMethodId?: string | null    // takes precedence over the merchant default
    metadata?: Record<string, unknown>
  }
}): Promise<Branch>
commerce.branches.archive({ id: string }): Promise<Branch>
```

---

## `fulfillment`

`fulfillment` exists only when at least one fulfillment adapter is configured. See [50-adapter-system.md](./50-adapter-system.md).

### `fulfillment.types.list`

Returns the full fulfillment type registry — core types plus any plugin or inline contributions. Used by admin UIs building the "create fulfillment method" form.

```ts
commerce.fulfillment.types.list(): Promise<{
  id: string                       // 'pickup' or 'restaurant:dinein'
  description: string | null
  dataSchema: JsonSchema           // serializable form of the Zod schema
}[]>
```

See [52-fulfillment-types.md](./52-fulfillment-types.md) for registration semantics.

### `fulfillment.methods.list`

```ts
commerce.fulfillment.methods.list({
  where?: {
    adapterId?: string
    type?: 'shipping' | 'delivery' | 'pickup' | 'digital'
    enabled?: boolean
  }
}): Promise<ListResult<FulfillmentMethod>>
```

### `fulfillment.methods.get`

```ts
commerce.fulfillment.methods.get({ id: string }): Promise<FulfillmentMethod>
```

### `fulfillment.methods.create`

```ts
commerce.fulfillment.methods.create({
  adapterId: string   // typed as union of registered fulfillment adapter keys
  type: string        // a registered fulfillment type ID; the adapter must handle it
  name: string
  enabled?: boolean
  pricing: FulfillmentPricing
  settings?: Record<string, unknown>
}): Promise<FulfillmentMethod>
```

`type` must be a value from the fulfillment type registry, and the chosen `adapterId` must declare support for it. Validation rejects mismatches at write time.

### `fulfillment.methods.update`

```ts
commerce.fulfillment.methods.update({
  id: string
  data: {
    name?: string
    enabled?: boolean
    pricing?: FulfillmentPricing
    settings?: Record<string, unknown>
  }
}): Promise<FulfillmentMethod>
```

### `fulfillment.methods.archive`

```ts
commerce.fulfillment.methods.archive({ id: string }): Promise<FulfillmentMethod>
```

### `fulfillment.track`

```ts
commerce.fulfillment.track({ orderId: string }): Promise<FulfillmentTracking>
```

### `fulfillment.createLabel`

```ts
commerce.fulfillment.createLabel({ orderId: string }): Promise<FulfillmentLabel>
```

---

## `delivery`

`delivery` exists only when the `deliveries: [...]` slot has at least one adapter. See [54-delivery-adapters.md](./54-delivery-adapters.md).

`commerce.delivery.methods.*` is automatically scoped to the request's branch context via `withContext` — listing methods returns only those visible at the current branch (own branch + merchant-wide + platform-wide). Creating without an explicit `branch` defaults to the request's branch.

### `delivery.methods.list`

Admin-style listing. Returns all visible methods regardless of cart or address.

```ts
commerce.delivery.methods.list({
  where?: {
    adapter?: string         // typed as union of registered delivery adapter IDs
    enabled?: boolean
    branch?: string | null   // explicit override; defaults to request context
  }
  limit?: number
  cursor?: string
}): Promise<ListResult<DeliveryMethod>>
```

### `delivery.methods.get`

```ts
commerce.delivery.methods.get({ id: string }): Promise<DeliveryMethod>
```

### `delivery.methods.create`

```ts
commerce.delivery.methods.create({
  adapter: string             // typed as union of registered delivery adapter IDs
  name: string
  enabled?: boolean
  branch?: string             // defaults to request's branch context
  pricing: PricingValue       // discriminated union from deliveryPricing strategies
  minOrderAmount?: Money
  maxDistanceMeters?: number
  autoDispatch?: 'inherit' | 'auto' | 'manual'   // default 'inherit' — uses merchant/app setting
  metadata?: Record<string, unknown>
}): Promise<DeliveryMethod>
```

`pricing` is built by calling a strategy factory: `distanceBased({...})`, `flat({...})`, etc. TS infers the settings shape from the strategy tag. The chosen strategy must be registered in `deliveryPricing: []`.

### `delivery.methods.update`

```ts
commerce.delivery.methods.update({
  id: string
  data: {
    name?: string
    enabled?: boolean
    pricing?: PricingValue
    minOrderAmount?: Money | null
    maxDistanceMeters?: number | null
    autoDispatch?: 'inherit' | 'auto' | 'manual'
    metadata?: Record<string, unknown>
  }
}): Promise<DeliveryMethod>
```

### `delivery.methods.archive`

```ts
commerce.delivery.methods.archive({ id: string }): Promise<DeliveryMethod>
```

### `delivery.methods.quote`

Checkout-time discovery. Runs every active method's strategy against the supplied cart and address and returns the priced + available options.

```ts
commerce.delivery.methods.quote({
  cart: Cart
  deliveryAddress: Address
  branch?: string             // defaults to request's branch context
}): Promise<Array<{
  methodId: string
  name: string
  adapter: string
  fee: Money
  etaSeconds?: number
  unavailable?:
    | 'below_min_amount'
    | 'out_of_range'
    | 'no_zone_match'
    | 'provider_unavailable'
    | 'geocoding_failed'
}>>
```

Methods where the strategy throws an unservable error are returned with `unavailable` populated and `fee` set to the strategy's last computed value (or zero). Methods that succeed have `unavailable: undefined`. Callers filter by `unavailable === undefined` to render eligible options.

This is a separate operation from `methods.list` because list is admin-oriented (no cart/address) and quote is checkout-oriented (requires both).

### `delivery.create`

Explicit dispatch. Usually triggered automatically by core on `orders:confirmed` (see [54-delivery-adapters.md](./54-delivery-adapters.md)); manual call is for admin tooling or apps that disable auto-dispatch.

```ts
commerce.delivery.create({
  orderId: string
  methodId?: string           // overrides order's deliveryMethodId
}): Promise<Delivery>
```

By default the operation reads the order's `deliveryMethodId` and dispatches via that method's adapter. Passing `methodId` updates the order's `deliveryMethodId` first, then dispatches — one round trip for the re-dispatch case. The order's method update and the dispatch run inside the same transaction; if dispatch fails, the method change rolls back.

### `delivery.cancel`

Capability-gated. Only present on the typed surface when at least one configured delivery adapter declares `capabilities.cancellation: true`. Calling on a delivery whose adapter doesn't support cancellation throws `CommerceCapabilityError` at runtime.

```ts
commerce.delivery.cancel({
  id: string
  reason?: string
}): Promise<Delivery>
```

### `delivery.get`

```ts
commerce.delivery.get({ id: string }): Promise<Delivery>
```

### `delivery.list`

```ts
commerce.delivery.list({
  where?: {
    orderId?: string
    methodId?: string
    adapter?: string
    state?: 'pending' | 'dispatched' | 'in_transit' | 'delivered' | 'cancelled' | 'failed'
  }
  orderBy?: { createdAt?: 'asc' | 'desc' }[]
  limit?: number
  cursor?: string
}): Promise<ListResult<Delivery>>
```

### `delivery.track`

Capability-gated. Only present on the typed surface when at least one configured delivery adapter declares `capabilities.realtimeTracking: true`. Returns the current driver and ETA snapshot.

```ts
commerce.delivery.track({ id: string }): Promise<DeliveryStatus>

type DeliveryStatus = {
  state: 'pending' | 'dispatched' | 'in_transit' | 'delivered' | 'cancelled' | 'failed'
  driverInfo?: { name: string; phone: string; vehicle?: string }
  location?: Coordinates
  estimatedArrival?: Date
  lastUpdate: Date
}
```

### `delivery.transitions`

The append-only audit log for a delivery — every state change with its source (`'core'`, `'webhook'`, `'poll'`) and timestamp.

```ts
commerce.delivery.transitions({ id: string }): Promise<DeliveryTransition[]>

type DeliveryTransition = {
  id: string
  deliveryId: string
  from: DeliveryState | null   // null for the initial transition
  to: DeliveryState
  source: 'core' | 'webhook' | 'poll'
  metadata?: Record<string, unknown>
  occurredAt: Date
}
```

### `delivery.strategies.list`

Returns the registered delivery pricing strategies. Used by admin UIs building the "create delivery method" form to render the right inputs per strategy.

```ts
commerce.delivery.strategies.list(): Promise<Array<{
  tag: string                  // e.g. 'distance-based' or 'app:vip-tier'
  description: string | null
  settingsSchema: JsonSchema   // serialized Zod schema for the strategy's settings
}>>
```

See [55-delivery-pricing.md](./55-delivery-pricing.md) for strategy registration.

---

## `calculation`

`calculation` exists only when `calculation.runtime: true` is set in `createCommerce()`. See [35-calculation-engine.md](./35-calculation-engine.md).

```ts
commerce.calculation.getPipeline({
  merchantId: string
}): Promise<{ pipeline: string[] } | null>

commerce.calculation.setPipeline({
  merchantId: string
  pipeline: string[]
}): Promise<void>

commerce.calculation.resetPipeline({
  merchantId: string
}): Promise<void>
```

Pipelines are validated against the step registry at write time — unknown step IDs throw `CommerceValidationError`.

---

## `tasks`

See [43-background-tasks.md](./43-background-tasks.md) for the full tasks API.

```ts
commerce.tasks.run(key: TaskKey, opts?: Record<string, unknown>): Promise<void>
commerce.tasks.list(): TaskDefinition[]
```

`TaskKey` is the union of all registered task keys from core and installed plugins, inferred at `createCommerce()` time.

---

## `metadata`

Introspection of the currently-installed configuration. Useful for admin UIs, generated API documentation, and `commerce doctor` tooling. All metadata calls are side-effect-free.

```ts
commerce.metadata.plugins(): Array<{
  id: string
  version: string
  support: { merchants: 'optional' | 'required' | 'forbidden'; branches: ... }
  operations: string[]                                  // operation keys contributed
  hooks: string[]                                       // hook keys subscribed
  states: string[]                                      // order states contributed
  tables: string[]                                      // table names contributed
  calculationSteps: string[]                            // step ids contributed
}>

commerce.metadata.operation(key: string): {
  source: 'core' | { plugin: string; version: string }
  input: JsonSchema                                     // Standard Schema → JSON Schema
  output?: JsonSchema
} | null

commerce.metadata.adapters(): {
  payments:     Array<{ id: string; capabilities: Record<string, boolean> }>
  deliveries:   Array<{ id: string; capabilities: Record<string, boolean> }>
  fulfillments: Array<{ id: string; capabilities: Record<string, boolean> }>
  payouts:      Array<{ id: string; capabilities: Record<string, boolean> }>
  storage:      Array<{ id: string }>
  scheduler:    { id: string } | null
}
```

All schema fields are emitted as JSON Schema (the Standard Schema → JSON Schema conversion is part of the contract — see [47-validation.md](./47-validation.md)) so admin UIs can render dynamic forms generically.

---

## Error behavior

All server SDK methods throw on failure. The thrown error is always a `CommerceError` subclass. See [15-errors.md](./15-errors.md) for the full hierarchy and codes.

```ts
import { CommerceNotFoundError, CommerceStateError } from 'commerce-kit'

try {
  await commerce.orders.confirm({ id: 'ord_123' })
} catch (err) {
  if (err instanceof CommerceStateError) {
    // order is not in a confirmable state
  }
  throw err
}
```

---

## Cross-links

- Entity shapes (`Product`, `Order`, `Payment`, etc.): [20-data-model.md](./20-data-model.md)
- Pricing pipeline used by `calculate` and `checkout`: [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Hook context and `ctx.operation` keys: [42-hooks.md](./42-hooks.md)
- HTTP route mapping for each method: [80-framework-adapters.md](./80-framework-adapters.md)
- Client SDK mirroring this surface: [80-framework-adapters.md](./80-framework-adapters.md)
- Type inference from `createCommerce()`: [60-type-inference.md](./60-type-inference.md)

## Future RFCs

- Bulk operations (`products.bulkUpdate`, `orders.bulkCancel`)
- `commerce.withContext` scope helpers for common role patterns
