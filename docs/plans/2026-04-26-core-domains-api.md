# Core Domains API Specification

> Superseded note: the canonical core domain API direction has moved into `docs/architecture/20-data-model.md`, `docs/architecture/60-type-inference.md`, `docs/architecture/80-framework-adapters.md`, and `docs/examples/getting-started.md`. Those docs now use the domain-first, object-only API with `orders.checkout(...)`, unified fulfillment, and high-level order payment workflows.

**Date:** 2026-04-26

## Overview

This document defines the full API contract for each core domain built into Commerce Kit.

## Query Contracts

All list operations share the same query structure:

```ts
type ListQuery<TWhere, TOrderBy> = {
  where?: TWhere
  orderBy?: TOrderBy[]
  limit?: number
  cursor?: string
}

type PageResult<T> = {
  items: T[]
  pageInfo: {
    nextCursor: string | null
    hasMore: boolean
  }
}
```

---

## Products

**Namespace:** `commerce.products`

### list

```ts
type ProductWhere = {
  id?: string | { in?: string[] }
  status?: "draft" | "active" | "archived"
  categoryId?: string | { in?: string[] }
  price?: { eq?: number; gt?: number; gte?: number; lt?: number; lte?: number }
  title?: { contains?: string }
  createdAt?: { gt?: string; gte?: string; lt?: string; lte?: string }
}

type ProductOrderBy =
  | { createdAt: "asc" | "desc" }
  | { updatedAt: "asc" | "desc" }
  | { title: "asc" | "desc" }
  | { price: "asc" | "desc" }

commerce.products.list({
  where: { status: "active", categoryId: { in: ["cat_1", "cat_2"] } },
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
  cursor: "next_cursor",
}) => Promise<PageResult<Product>>
```

### get

```ts
commerce.products.get(id: string) => Promise<Product>
```

### create

```ts
type CreateProductInput = {
  title: string
  slug: string
  description?: string
  price?: number
  compareAtPrice?: number
  status?: "draft" | "active"
  categoryId?: string
  images?: string[]
  variants?: CreateVariantInput[]
  metadata?: Record<string, unknown>
}

commerce.products.create(input: CreateProductInput) => Promise<Product>
```

### update

```ts
type UpdateProductInput = Partial<CreateProductInput>

commerce.products.update(id: string, input: UpdateProductInput) => Promise<Product>
```

### archive

```ts
commerce.products.archive(id: string) => Promise<Product>
```

### versions

```ts
type ProductVersionWhere = {
  version?: number
  changedAt?: { gt?: string; gte?: string; lt?: string; lte?: string }
}

type ProductVersionOrderBy =
  | { version: "asc" | "desc" }
  | { changedAt: "asc" | "desc" }

commerce.products.versions(id: string, {
  where: {},
  orderBy: [{ version: "desc" }],
}) => Promise<PageResult<ProductVersion>>
```

### getVersion

```ts
commerce.products.getVersion(id: string, version: number) => Promise<ProductVersion>
```

---

## Orders

**Namespace:** `commerce.orders`

### list

```ts
type OrderWhere = {
  id?: string | { in?: string[] }
  status?: OrderStatus
  customerId?: string | { in?: string[] }
  paymentStatus?: "pending" | "authorized" | "captured" | "failed" | "refunded" | "partially_refunded"
  createdAt?: { gt?: string; gte?: string; lt?: string; lte?: string }
}

type OrderOrderBy =
  | { createdAt: "asc" | "desc" }
  | { updatedAt: "asc" | "desc" }
  | { total: "asc" | "desc" }

commerce.orders.list({
  where: { status: "confirmed", customerId: "cust_123" },
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
}) => Promise<PageResult<Order>>
```

### get

```ts
commerce.orders.get(id: string) => Promise<Order>
```

### create

```ts
type CreateOrderInput = {
  customerId: string
  lines: {
    productId: string
    variantId?: string
    quantity: number
    unitPrice: number
  }[]
  shippingAddress?: Address
  billingAddress?: Address
  paymentAdapterId?: string
  metadata?: Record<string, unknown>
}

commerce.orders.create(input: CreateOrderInput) => Promise<Order>
```

### calculate

```ts
type CalculateOrderInput = Omit<CreateOrderInput, "paymentAdapterId">

commerce.orders.calculate(input: CalculateOrderInput) => Promise<{
  subtotal: number
  taxTotal: number
  discountTotal: number
  total: number
  lines: { productId: string; quantity: number; unitPrice: number; total: number }[]
}>
```

### transition

```ts
type TransitionOrderInput = {
  to: OrderStatus
  metadata?: Record<string, unknown>
}

commerce.orders.transition(id: string, input: TransitionOrderInput) => Promise<Order>
```

### split

Split an order into multiple fulfillments:

```ts
type SplitOrderInput = {
  lines: {
    lineId: string
    quantity: number
    shippingAddress?: Address
    shippingMethod?: string
  }[]
}

commerce.orders.split(orderId: string, input: SplitOrderInput) => Promise<Order[]>
```

---

## Payments

**Namespace:** `commerce.payments`

### list

```ts
type PaymentWhere = {
  id?: string | { in?: string[] }
  orderId?: string | { in?: string[] }
  status?: "pending" | "authorized" | "captured" | "failed" | "refunded" | "partially_refunded" | "cancelled"
  providerReference?: string
  createdAt?: { gt?: string; gte?: string; lt?: string; lte?: string }
}

type PaymentOrderBy =
  | { createdAt: "asc" | "desc" }
  | { amount: "asc" | "desc" }

commerce.payments.list({
  where: { orderId: "order_123" },
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
}) => Promise<PageResult<Payment>>
```

### get

```ts
commerce.payments.get(id: string) => Promise<Payment>
```

---

## Customers

**Namespace:** `commerce.customers`

### list

```ts
type CustomerWhere = {
  id?: string | { in?: string[] }
  email?: { eq?: string; contains?: string }
  createdAt?: { gt?: string; gte?: string; lt?: string; lte?: string }
}

type CustomerOrderBy =
  | { createdAt: "asc" | "desc" }
  | { email: "asc" | "desc" }

commerce.customers.list({
  where: { email: { contains: "@example.com" } },
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
}) => Promise<PageResult<Customer>>
```

### get

```ts
commerce.customers.get(id: string) => Promise<Customer>
```

### create

```ts
type CreateCustomerInput = {
  email: string
  firstName?: string
  lastName?: string
  phone?: string
  metadata?: Record<string, unknown>
}

commerce.customers.create(input: CreateCustomerInput) => Promise<Customer>
```

### update

```ts
type UpdateCustomerInput = Partial<CreateCustomerInput>

commerce.customers.update(id: string, input: UpdateCustomerInput) => Promise<Customer>
```

---

## Categories

**Namespace:** `commerce.categories`

### list

```ts
type CategoryWhere = {
  id?: string | { in?: string[] }
  parentId?: string | { eq?: null } | { in?: string[] }
  name?: { contains?: string }
  slug?: string
  createdAt?: { gt?: string; gte?: string; lt?: string; lte?: string }
}

type CategoryOrderBy =
  | { createdAt: "asc" | "desc" }
  | { name: "asc" | "desc" }
  | { sortOrder: "asc" | "desc" }

commerce.categories.list({
  where: { parentId: { eq: null } },
  orderBy: [{ sortOrder: "asc" }],
  limit: 50,
}) => Promise<PageResult<Category>>
```

### get

```ts
commerce.categories.get(id: string) => Promise<Category>
```

### create

```ts
type CreateCategoryInput = {
  name: string
  slug: string
  description?: string
  parentId?: string | null
  sortOrder?: number
  metadata?: Record<string, unknown>
}

commerce.categories.create(input: CreateCategoryInput) => Promise<Category>
```

### update

```ts
type UpdateCategoryInput = Partial<CreateCategoryInput>

commerce.categories.update(id: string, input: UpdateCategoryInput) => Promise<Category>
```

### archive

```ts
commerce.categories.archive(id: string) => Promise<Category>
```

---

## Cart

**Namespace:** `commerce.cart`

### get

```ts
commerce.cart.get(cartId: string) => Promise<Cart>
```

### create

```ts
type CreateCartInput = {
  customerId?: string
  lines?: {
    productId: string
    variantId?: string
    quantity: number
  }[]
  metadata?: Record<string, unknown>
}

commerce.cart.create(input?: CreateCartInput) => Promise<Cart>
```

### update

```ts
type UpdateCartInput = {
  customerId?: string
  lines?: {
    productId: string
    variantId?: string
    quantity: number
  }[]
  metadata?: Record<string, unknown>
}

commerce.cart.update(cartId: string, input: UpdateCartInput) => Promise<Cart>
```

### delete

```ts
commerce.cart.delete(cartId: string) => Promise<void>
```

---

## Shared Types

```ts
type Address = {
  firstName: string
  lastName: string
  line1: string
  line2?: string
  city: string
  state?: string
  postalCode: string
  country: string
  phone?: string
}

type ProductVersion = {
  version: number
  title: string
  slug: string
  description?: string
  price?: number
  compareAtPrice?: number
  categoryId?: string
  images: string[]
  variants: Omit<Variant, "id">[]
  changedAt: string
  changedBy?: string
}

type Product = {
  id: string
  title: string
  slug: string
  description?: string
  price?: number
  compareAtPrice?: number
  status: "draft" | "active" | "archived"
  categoryId?: string
  images: string[]
  variants: Variant[]
  metadata: Record<string, unknown>
  version: number
  versions: ProductVersion[]
  createdAt: string
  updatedAt: string
}

type Order = {
  id: string
  orderNumber: string
  customerId: string
  status: OrderStatus

  // Fulfillment: parent order ID (null for original orders, set for split fulfillments)
  parentId?: string | null

  lines: OrderLine[]
  subtotal: number
  taxTotal: number
  discountTotal: number
  shippingTotal: number
  total: number
  currency: string
  paymentStatus: PaymentStatus
  paymentAdapterId?: string
  paymentReference?: string
  shippingAddress?: Address
  billingAddress?: Address
  shippingMethod?: string
  trackingNumber?: string
  carrier?: string
  metadata: Record<string, unknown>
  pluginData: Record<string, unknown>
  createdAt: string
  updatedAt: string
}

type Customer = {
  id: string
  email: string
  firstName?: string
  lastName?: string
  phone?: string
  metadata: Record<string, unknown>
  createdAt: string
  updatedAt: string
}

type Category = {
  id: string
  name: string
  slug: string
  description?: string
  parentId?: string | null
  sortOrder: number
  metadata: Record<string, unknown>
  createdAt: string
  updatedAt: string
}

type Cart = {
  id: string
  customerId?: string
  lines: CartLine[]
  subtotal: number
  metadata: Record<string, unknown>
  createdAt: string
  updatedAt: string
}
```

---

## Notes

- `products.archive` is a soft delete (sets status to archived)
- `categories.archive` is a soft delete (sets status to archived)
- `orders.calculate` is a read-only simulation that returns computed totals without creating an order
- `cart` uses a session-bound ID rather than a user ID for anonymous carts
- All IDs are opaque strings (no specific format enforced)
- All timestamps are ISO 8601 strings
