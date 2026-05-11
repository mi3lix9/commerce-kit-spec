# Adapter System

## Purpose

This document defines the Commerce Kit adapter model: packaging boundaries, database and provider adapter interfaces, dormant activation rules, and the unified fulfillment interface.

Plugin extension rules live in [40-plugin-system.md](./40-plugin-system.md). Compile-time surfacing of adapters and plugin-gated APIs lives in [60-type-inference.md](./60-type-inference.md).

## Non-goals

- Repeating plugin dependency rules
- Repeating framework-specific HTTP mounting details
- Re-specifying the core data model except where activation affects schema ownership

## Core decisions

- Commerce Kit bundles core adapter interfaces and the `drizzleAdapter` in core.
- Provider implementations ship as separate packages.
- Database and payment adapters are core contracts; database is required and payment is required.
- Fulfillment, storage, and payout follow an optional dormant activation model.
- Dormant activation is owned by `createCommerce()`, not by plugins.

## Config shape

All adapter slots use objects. Object keys are the adapter IDs and are inferred as literal types by TypeScript, enabling compile-time narrowing (e.g. the `adapterId` field at checkout is typed as the union of registered payment adapter keys).

```ts
export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),

  payment: {
    moyasar: moyasar({ secretKey: env.MOYASAR_SECRET }),
    tabby: tabby({ apiKey: env.TABBY_API_KEY }),
  },

  fulfillment: {
    shipping: shippo({ apiKey: env.SHIPPO_API_KEY }),
    riyadhDelivery: localDelivery(),
    pickup: storePickup(),
  },

  storage: {
    media: r2({ bucket: env.R2_BUCKET, accountId: env.CF_ACCOUNT_ID }),
  },

  payout: {
    hyperpay: hyperpay({ apiKey: env.HYPERPAY_API_KEY }),
  },

  scheduler: {
    jobs: bullmq({ redis: env.REDIS_URL }),
  },

  plugins: [coupons(), marketplace()],
})
```

`database` is a single required adapter (not an object) because only one database adapter is active at a time. All optional adapter slots (`payment`, `fulfillment`, `storage`, `payout`, `scheduler`) accept objects and are dormant when omitted.

## Packaging model

### Bundled in `commerce-kit`

- `drizzleAdapter`
- adapter interfaces for `payment`, `fulfillment`, `storage`, and `payout`
- `definePlugin`
- `createCommerce`

Core does not own the fetch client runtime package. The canonical client factory is published from `@commerce-kit/client`, while core remains the source of server-instance types that the client package consumes.

### Separate packages

Provider implementations and feature plugins ship separately. The canonical package inventory lives in [70-packages-and-monorepo.md](./70-packages-and-monorepo.md).

Tax is not a bundled adapter. Tax is handled by the pricing rules plugin as a pricing rule of type `tax`.

## Database adapter

The database adapter is required. Commerce Kit reads and writes data only through this interface.

```ts
interface DatabaseAdapter {
  findOne<T>(table: string, query: WhereClause): Promise<T | null>
  findMany<T>(table: string, query: QueryOptions): Promise<T[]>
  create<T>(table: string, data: Partial<T>): Promise<T>
  update<T>(table: string, query: WhereClause, data: Partial<T>): Promise<T>
  delete(table: string, query: WhereClause): Promise<void>
  count(table: string, query: WhereClause): Promise<number>
  increment(table: string, query: WhereClause, field: string, by: number): Promise<void>
  decrement(table: string, query: WhereClause, field: string, by: number): Promise<void>
  transaction<T>(fn: (tx: DatabaseAdapter) => Promise<T>): Promise<T>
}
```

Normative rules:

- `increment` and `decrement` must be single atomic database operations.
- Commerce Kit code and plugins must not bypass the adapter contract inside `transaction()`.
- In v1, `drizzleAdapter` is bundled and supports PostgreSQL, MySQL, and SQLite.
- Plugin hooks and plugin-owned API handlers use this same adapter contract and share the enclosing Commerce Kit transaction boundary for a write operation unless documented otherwise.

## Payment adapter

Payment is a core required interface. Multiple payment adapters may be registered at once, keyed by their config key, and routed by that key at checkout.

```ts
interface PaymentAdapter {
  capabilities: PaymentCapabilities
  authorize(ctx: AuthorizeContext): Promise<AuthorizeResult>
  capture(providerReference: string, amount: number): Promise<CaptureResult>
  refund(providerReference: string, amount: number, reason?: string): Promise<RefundResult>
  cancel(providerReference: string): Promise<CancelResult>
  verifyWebhook(payload: unknown, signature: string): Promise<WebhookEvent>
  supportsOffSession?: true
  chargeOffSession?(customerId: string, amount: number, currency: string): Promise<OffSessionResult>
}
```

Not every provider supports every payment flow; capabilities are explicit.

### Multiple payment adapters

```ts
createCommerce({
  payment: {
    moyasar: moyasar({ secretKey: env.MOYASAR_SECRET }),
    tabby: tabby({ apiKey: env.TABBY_API_KEY }),
  },
})
```

The config key is the adapter ID. TypeScript infers the literal union `'moyasar' | 'tabby'` from the object keys, so passing an unknown `adapterId` at checkout is a compile error.

At checkout, the caller selects the adapter by key:

```ts
await commerce.orders.checkout({
  items: cart.items,
  payment: { adapterId: 'moyasar' },
})
```

Webhook URLs follow the same key convention: `/webhooks/payment/moyasar`, `/webhooks/payment/tabby`.

The payment persistence model distinguishes between:

- transaction/operation references used by adapter methods such as `capture`, `refund`, and `cancel`
- inbound provider event identifiers used for webhook idempotency

These references may be different values and must not be conflated.

Payment adapter methods are core operational APIs. Their HTTP exposure, if any, is defined by the framework adapter contract rather than implied directly by the adapter interface.

## Dormant activation model

Fulfillment, storage, payout, and scheduler interfaces are defined in core but are optional. If a given interface has no registered provider adapter, it is dormant.

For a dormant interface:

- no schema owned by that interface is created
- no fulfillment-specific states are registered
- no routes are mounted
- no lifecycle hooks for that interface run
- no interface-specific TypeScript surface exists

When at least one provider adapter is registered, `createCommerce()` activates the interface and materializes its schema, states, routes, and typed API surface.

That materialized schema is part of the Commerce Kit-managed generation surface described in `90-cli-and-tooling.md`.

The compile-time removal of dormant surface area is specified in [60-type-inference.md](./60-type-inference.md).

## Fulfillment adapter

Fulfillment is optional and dormant until configured via `createCommerce({ fulfillment: { ... } })`.

Fulfillment is the public abstraction for carrier shipping, local delivery, pickup, and digital delivery. The public API is unified even when provider adapters have different capabilities.

```ts
interface FulfillmentAdapter {
  types: FulfillmentMethodType[]
  capabilities: FulfillmentCapabilities

  listMethods?(ctx: ListFulfillmentMethodsContext): Promise<FulfillmentMethodCandidate[]>
  createFulfillment?(ctx: CreateFulfillmentContext): Promise<CreateFulfillmentResult>
  cancelFulfillment?(ctx: CancelFulfillmentContext): Promise<CancelFulfillmentResult>
  track?(ctx: TrackFulfillmentContext): Promise<FulfillmentTracking>
  createLabel?(ctx: CreateLabelContext): Promise<FulfillmentLabel>
  verifyWebhook?(payload: unknown, signature: string): Promise<FulfillmentWebhookEvent>
}

type FulfillmentMethodType = "shipping" | "delivery" | "pickup" | "digital"

interface FulfillmentCapabilities {
  rates?: boolean
  labels?: boolean
  tracking?: boolean
  pickupWindows?: boolean
  deliveryWindows?: boolean
  proofOfDelivery?: boolean
  digitalDelivery?: boolean
}
```

Config uses an object. The key is the adapter ID, inferred as a literal type. Multiple instances of the same adapter factory are supported naturally:

```ts
createCommerce({
  fulfillment: {
    shipping: shippo({ apiKey: env.SHIPPO_API_KEY }),
    riyadhDelivery: localDelivery(),
    jeddahDelivery: localDelivery(),
    pickup: storePickup(),
  },
})
```

When fulfillment is active:

- Commerce Kit materializes fulfillment method and fulfillment record persistence.
- Merchant-configured fulfillment methods, settings, and pricing are runtime database records.
- Config registers adapter code and capabilities only; it is not a runtime source of fulfillment pricing.
- Commerce Kit exposes `fulfillment.methods.*` for runtime method management and checkout method listing.
- Order checkout selects a configured fulfillment method by ID.

### Fulfillment method pricing

Core includes common fulfillment pricing strategies by default:

```ts
type FulfillmentPricing =
  | { type: "free" }
  | { type: "flat"; price: Money }
  | { type: "threshold"; price: Money; freeAfter: Money }
  | { type: "distance"; minimum: Money; perKm: Money }
  | { type: "adapter" }
```

Example runtime method:

```ts
await client.fulfillment.methods.create({
  adapter: "localDelivery",
  type: "delivery",
  name: "Riyadh Delivery",
  enabled: true,
  pricing: {
    type: "distance",
    minimum: { amount: 1700, currency: "SAR" },
    perKm: { amount: 100, currency: "SAR" },
  },
})
```

Advanced plugins may add pricing strategies later, but normal users should not need to configure a strategy registry.

## Storage adapter

Storage is an optional interface and follows the same dormant activation pattern.

```ts
interface StorageAdapter {
  upload(ctx: UploadContext): Promise<UploadResult>
  delete(key: string): Promise<void>
  getUrl(key: string): Promise<string>
  getSignedUrl?(key: string, expiresIn: number): Promise<string>
}

type UploadContext = {
  key: string
  body: Buffer | ReadableStream
  contentType: string
  metadata?: Record<string, string>
}

type UploadResult = {
  key: string
  url: string
  size: number
}
```

Config:

```ts
createCommerce({
  storage: {
    media: r2({ bucket: env.R2_BUCKET, accountId: env.CF_ACCOUNT_ID }),
  },
})
```

Storage is used by core for product media and by the optional digital products plugin for delivery assets. `getSignedUrl` is optional and only required when the adapter needs time-limited access URLs (e.g. private digital product downloads).

## Payout adapter

Payout is an optional interface bundled in core and activated only when a payout provider is registered.

The payout interface exists so marketplace-oriented plugins can request provider-neutral payout behavior without moving payout orchestration into core business logic.

```ts
interface PayoutAdapter {
  capabilities: PayoutCapabilities
  createPayout(ctx: CreatePayoutContext): Promise<CreatePayoutResult>
  cancelPayout(payoutReference: string): Promise<CancelPayoutResult>
  getPayoutStatus(payoutReference: string): Promise<PayoutStatus>
  verifyWebhook(payload: unknown, signature: string): Promise<PayoutWebhookEvent>
}
```

Config:

```ts
createCommerce({
  payout: {
    hyperpay: hyperpay({ apiKey: env.HYPERPAY_API_KEY }),
  },
})
```

Normative rules:

- Payout activates only when a payout provider is registered.
- Payout activation is commonly required by `@commerce-kit/vendor-payouts`, but activation itself is still owned by core.
- Payout-specific schema, routes, hooks, and typed APIs remain dormant until activation.

## Scheduler adapter

The scheduler adapter is optional and follows the same dormant activation pattern. When not configured, deferred task scheduling is dormant and no task-related schema or behavior is active.

```ts
interface SchedulerAdapter {
  schedule(task: ScheduledTask): Promise<void>
  cancel(taskId: string): Promise<void>
}

type ScheduledTask = {
  id: string       // idempotency key
  type: string     // registered task key, e.g. 'orders:auto-cancel'
  data: unknown
  runAt: Date
  retries?: number
}
```

Config:

```ts
createCommerce({
  scheduler: {
    jobs: bullmq({ redis: env.REDIS_URL }),
  },
})
```

Normative rules:

- Scheduling the same `task.id` twice must be a no-op on the adapter side.
- `cancel` must be safe to call for a task that has already run or does not exist.
- The scheduler adapter does not own task execution logic — Commerce Kit owns that through `commerce.tasks`.

For the full deferred task and recurring task model, see [43-background-tasks.md](./43-background-tasks.md).

## Future RFCs

- Additional optional adapter interfaces beyond the current core set
- Capability model changes for existing adapter contracts
