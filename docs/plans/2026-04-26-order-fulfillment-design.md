# Order Fulfillment Design

> Superseded note: the canonical fulfillment direction has moved into `docs/architecture/20-data-model.md`, `docs/architecture/50-adapter-system.md`, `docs/architecture/60-type-inference.md`, and `docs/examples/getting-started.md`. Those docs now use unified fulfillment instead of separate shipping/delivery public APIs, and checkout is modeled with `orders.checkout(...)`.

**Date:** 2026-04-26

## Overview

Fulfillment orders are just regular orders with a `parentId` reference. No separate state and no separate fulfillment lifecycle. The only extra public API needed is `orders.split(...)`.

## Database Design

Single orders table:

```ts
type Order = {
  id: string
  orderNumber: string

  // Parent-child relationship (self-referential)
  parentId?: string | null

  // Order status
  status: OrderStatus

  // Customer
  customerId: string

  // Lines
  lines: OrderLine[]

  // Money
  subtotal: number
  taxTotal: number
  discountTotal: number
  shippingTotal: number
  total: number
  currency: string

  // Payment
  paymentStatus: PaymentStatus

  // Shipping
  shippingAddress?: Address
  shippingMethod?: string
  trackingNumber?: string
  carrier?: string

  // Meta
  metadata: Record<string, unknown>
  createdAt: string
  updatedAt: string
}
```

**Key points:**
- Original orders have `parentId: null`
- Split/fulfillment orders have `parentId: <original_order_id>`
- Same table, same structure

## API Design

Use the existing order API plus `orders.split(...)`.

### Create order

```ts
commerce.orders.create({ ... }) => Order
```

### Split order

```ts
// Split creates child orders with parentId set
commerce.orders.split(orderId, {
  lines: [
    { lineId: "line_1", quantity: 2, shippingAddress: addressB },
    { lineId: "line_1", quantity: 3, shippingAddress: addressC },
  ],
}) => Order[]
```

### Get child orders

```ts
// split() returns the child order IDs
const [fulfillment1, fulfillment2] = await commerce.orders.split(orderId, { ... })

// Get details by ID
await commerce.orders.get(fulfillment1.id)
await commerce.orders.get(fulfillment2.id)

// Or list by IDs
commerce.orders.list({
  where: { id: { in: [fulfillment1.id, fulfillment2.id] } }
}) => PageResult<Order>
```

### Get single order

```ts
commerce.orders.get(fulfillmentId) => Order
```

### Cancel fulfillment

```ts
// Same transition method
commerce.orders.transition(fulfillmentId, { to: "cancelled" }) => Order
```

## How it works

1. **Create normal order** — `orders.create({ ... })` → original order with `parentId: null`

2. **Split when needed** — `orders.split(orderId, { lines: [...] })` → creates child orders with `parentId: <original_order_id>`

3. **Track fulfillments** — `split()` returns child order IDs, then use `orders.get(id)` or `orders.list({ where: { id: { in: [...] } } })`

4. **Cancel** — use `orders.transition(fulfillmentId, { to: "cancelled" })`

That's it. No separate fulfillment states, no new methods.
