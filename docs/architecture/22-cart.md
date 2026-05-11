# Cart

## Purpose

Define the cart model, client-side state management, auto-recalculation behavior, server persistence rules, and the `orders.calculate` operation.

## Non-goals

- Order creation and checkout — see [20-data-model.md](./20-data-model.md) and [80-framework-adapters.md](./80-framework-adapters.md)
- Pricing pipeline details — see [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Cart expiry background task — see [43-background-tasks.md](./43-background-tasks.md)

## Core decisions

- The cart lives on the client by default, managed by `@commerce-kit/client`.
- Every cart mutation triggers an automatic `orders.calculate` call, keeping totals server-authoritative at all times.
- Server-side cart persistence is opt-in. When enabled, the cart is stored as an order record with a `cart` status on the orders table.
- `orders.calculate` is a pure read — it calculates totals for a given item set without creating or mutating any record.
- The server always re-validates prices at checkout regardless of what the client calculated.

## Client-side cart

The cart is managed in the `@commerce-kit/client` SDK in the browser. No server involvement until checkout.

### Storage

```ts
const client = createCommerceClient<typeof commerce>({
  baseUrl: '/api/commerce',
  cart: {
    storage: 'localStorage',  // 'localStorage' | 'cookie' | 'memory'
  },
})
```

### Cart shape

```ts
type Cart = {
  id: string
  items: CartItem[]
  subtotal: Money
  discountTotal: Money
  taxTotal: Money | null          // null until an address is provided
  fulfillmentTotal: Money | null  // null until a fulfillment method is selected
  total: Money
  currency: string
  appliedCoupon: AppliedCoupon | null
  updatedAt: Date
}

type CartItem = {
  id: string
  variantId: string
  quantity: number
  unitPrice: Money
  total: Money
  productTitle: string
  variantAttributes: Record<string, string>
}

type AppliedCoupon = {
  code: string
  discount: Money
}
```

### Operations

All cart mutation methods return the updated cart with recalculated totals. `cart.get()` returns the last known local state without triggering a server call.

```ts
// mutations — each auto-recalculates after the local state change
const cart = await client.cart.add({ variantId: 'var_123', quantity: 1 })
const cart = await client.cart.update({ itemId: 'ci_456', quantity: 3 })
const cart = await client.cart.remove({ itemId: 'ci_456' })
const cart = await client.cart.clear()

// explicit recalculation — use when options change without a cart mutation
const cart = await client.cart.calculate({
  couponCode: 'SAVE10',
  fulfillmentMethodId: 'method_shipping',
  address: { country: 'SA', city: 'Riyadh', postalCode: '12345' },
})

// synchronous local read — returns last known state, no server call
const cart = client.cart.get()
```

### Auto-recalculation

Every mutation (`add`, `update`, `remove`, `clear`) calls `orders.calculate` automatically after updating local state. The cart returned from a mutation always reflects server-calculated prices, discounts, and taxes.

The recalculation call carries forward the last known options (coupon, fulfillment method, address). Options are updated explicitly via `cart.calculate(...)`.

If `orders.calculate` fails, the mutation still applies to local state and the cart retains its previous totals. The SDK surfaces the error but does not roll back the local item change.

### Checkout

The client passes cart items directly to `orders.checkout`. The server re-calculates prices from scratch — client-side totals are display-only and are never trusted for order creation.

```ts
const cart = client.cart.get()

const result = await client.orders.checkout({
  items: cart.items.map(i => ({ variantId: i.variantId, quantity: i.quantity })),
  fulfillmentMethodId: 'method_shipping',
  payment: { providerId: 'moyasar' },
})
```

---

## `orders.calculate`

A pure read operation that applies the full pricing pipeline to a given item set and returns totals. No order is created or modified.

### HTTP route

```
POST /orders/calculate
```

### Request

```ts
type CalculateInput = {
  items: { variantId: string; quantity: number }[]
  couponCode?: string
  fulfillmentMethodId?: string
  address?: {
    country: string
    city?: string
    postalCode?: string
  }
}
```

### Response

```ts
type CalculateResult = {
  items: {
    variantId: string
    quantity: number
    unitPrice: Money
    total: Money
    productTitle: string
    variantAttributes: Record<string, string>
  }[]
  subtotal: Money
  discountTotal: Money
  taxTotal: Money | null
  fulfillmentTotal: Money | null
  total: Money
  currency: string
  appliedCoupon: AppliedCoupon | null
}
```

`taxTotal` is `null` when no address is provided or when the active tax rules require an address to calculate. `fulfillmentTotal` is `null` when no `fulfillmentMethodId` is provided.

### Server API

`orders.calculate` is also callable directly on the server instance:

```ts
const totals = await commerce.orders.calculate({
  items: [{ variantId: 'var_123', quantity: 2 }],
  couponCode: 'SAVE10',
})
```

---

## Server-side cart persistence (opt-in)

When enabled, the cart is stored on the orders table as an order with `cart` status. This allows carts to survive device changes and enables server-side cart management.

### Activation

```ts
const commerce = createCommerce({
  database: drizzleAdapter({ db }),
  payments: [moyasarPayments(...)],
  cart: {
    persistence: true,
    expiry: '7d',  // inactive carts are expired after this window
  },
})
```

When `cart.persistence` is `false` (default), no cart-related schema or routes are created.

### Client config for server-persisted carts

```ts
const client = createCommerceClient<typeof commerce>({
  baseUrl: '/api/commerce',
  cart: {
    storage: 'server',
  },
})
```

In `server` storage mode, all cart operations go over HTTP. The cart is identified by a cart ID stored in a cookie. Auto-recalculation still applies — the server returns updated totals on every mutation response.

### Checkout with a server-persisted cart

When the cart has a server-side record, checkout can reference it by ID:

```ts
await client.orders.checkout({
  cartId: cart.id,
  fulfillmentMethodId: 'method_shipping',
  payment: { providerId: 'moyasar' },
})
```

### `cart` order status

The `cart` status is added to the order state machine when server persistence is enabled. It is dormant otherwise.

```
cart → draft      (when checkout begins)
cart → cancelled  (when cart expires or is explicitly abandoned)
```

The `cart` status is not shown in normal order history. Orders with `cart` status are pre-checkout records and do not appear in `orders.list` results by default.

### HTTP routes (activated when `cart.persistence: true`)

```text
GET    /cart/:cartId
POST   /cart
POST   /cart/:cartId/items
PATCH  /cart/:cartId/items/:itemId
DELETE /cart/:cartId/items/:itemId
DELETE /cart/:cartId
```

---

## Cross-links

- Order state machine and base states: [20-data-model.md](./20-data-model.md)
- Pricing pipeline that `orders.calculate` runs: [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Cart expiry as a background task: [43-background-tasks.md](./43-background-tasks.md)
- Client SDK inference from server config: [60-type-inference.md](./60-type-inference.md)
- HTTP route mounting and framework adapters: [80-framework-adapters.md](./80-framework-adapters.md)
