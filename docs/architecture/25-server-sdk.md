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
})

// All calls on `c` pass this context to hooks
await c.orders.cancel({ id: 'ord_123' })
await c.products.archive({ id: 'prod_456' })
```

`withContext` returns a new SDK instance with the same shape. It does not mutate `commerce`.

```ts
interface RequestContext {
  customerId?: string | null
  actorId?: string | null
  actorType?: string
  roles?: string[]
  permissions?: string[]
  locale?: string
  metadata?: Record<string, unknown>
}
```

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
  payment: {
    adapterId: string            // typed as union of registered payment adapter keys
    metadata?: Record<string, unknown>  // adapter-specific fields
  }
  couponCode?: string
  metadata?: Record<string, unknown>
}): Promise<{
  order: Order
  paymentReference: string
  paymentUrl?: string            // present for redirect-based payment flows
}>
```

`paymentUrl` is returned when the payment adapter requires a redirect (e.g. a hosted payment page). When present, the caller must redirect the customer to complete payment. When absent, payment was handled in-process.

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

Marks the order as paid. Used for COD, offline, or manual payment flows where a webhook is not expected.

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
  amount?: Money    // partial refund when provided; full refund when omitted
  reason?: string
}): Promise<Order>
```

Triggers the payment adapter's `refund` internally and appends a record to the payment ledger.

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

## `fulfillment`

`fulfillment` exists only when at least one fulfillment adapter is configured. See [50-adapter-system.md](./50-adapter-system.md).

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
  type: 'shipping' | 'delivery' | 'pickup' | 'digital'
  name: string
  enabled?: boolean
  pricing: FulfillmentPricing
  settings?: Record<string, unknown>
}): Promise<FulfillmentMethod>
```

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

## `tasks`

See [43-background-tasks.md](./43-background-tasks.md) for the full tasks API.

```ts
commerce.tasks.run(key: TaskKey, opts?: Record<string, unknown>): Promise<void>
commerce.tasks.list(): TaskDefinition[]
```

`TaskKey` is the union of all registered task keys from core and installed plugins, inferred at `createCommerce()` time.

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
