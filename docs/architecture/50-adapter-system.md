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

Payment is a core required interface. Multiple payment adapters may be registered at once and routed by `paymentAdapterId`.

```ts
interface PaymentAdapter {
  id: string
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

The payment persistence model distinguishes between:

- transaction/operation references used by adapter methods such as `capture`, `refund`, and `cancel`
- inbound provider event identifiers used for webhook idempotency

These references may be different values and must not be conflated.

Payment adapter methods are core operational APIs. Their HTTP exposure, if any, is defined by the framework adapter contract rather than implied directly by the adapter interface.

## Dormant activation model

Fulfillment, storage, and payout interfaces are defined in core but are optional. If a given interface has no registered provider adapter, it is dormant.

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

Fulfillment is optional and dormant until configured via `createCommerce({ fulfillment: [...] })`.

Fulfillment is the public abstraction for carrier shipping, local delivery, pickup, and digital delivery. The public API is unified even when provider adapters have different capabilities.

```ts
interface FulfillmentAdapter {
  id: string
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

Config stays intentionally simple:

```ts
createCommerce({
  fulfillment: [
    localDelivery(),
    shippo(),
    storePickup(),
  ],
})
```

Adapters have default IDs. Advanced users can override IDs only when multiple instances of the same adapter are needed:

```ts
createCommerce({
  fulfillment: [
    localDelivery({ id: "riyadhDelivery" }),
    localDelivery({ id: "jeddahDelivery" }),
  ],
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

- Storage activates only when a storage provider is registered.
- The storage adapter contract is intentionally left minimal in the current architecture set because no v1 first-party storage package is defined.

## Payout adapter

Payout is an optional interface bundled in core and activated only when a payout provider is registered.

The payout interface exists so marketplace-oriented plugins can request provider-neutral payout behavior without moving payout orchestration into core business logic.

```ts
interface PayoutAdapter {
  id: string
  capabilities: PayoutCapabilities
  createPayout(ctx: CreatePayoutContext): Promise<CreatePayoutResult>
  cancelPayout(payoutReference: string): Promise<CancelPayoutResult>
  getPayoutStatus(payoutReference: string): Promise<PayoutStatus>
  verifyWebhook(payload: unknown, signature: string): Promise<PayoutWebhookEvent>
}
```

Normative rules:

- Payout activates only when a payout provider is registered.
- Payout activation is commonly required by `@commerce-kit/vendor-payouts`, but activation itself is still owned by core.
- Payout-specific schema, routes, hooks, and typed APIs remain dormant until activation.

## Scheduler adapter

The scheduler adapter is optional and follows the same dormant activation pattern. When not configured, deferred task scheduling is dormant and no task-related schema or behavior is active.

```ts
interface SchedulerAdapter {
  id: string
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

Normative rules:

- Scheduling the same `task.id` twice must be a no-op on the adapter side.
- `cancel` must be safe to call for a task that has already run or does not exist.
- The scheduler adapter does not own task execution logic — Commerce Kit owns that through `commerce.tasks`.

For the full deferred task and recurring task model, see [43-background-tasks.md](./43-background-tasks.md).

## Future RFCs

- Additional optional adapter interfaces beyond the current core set
- Capability model changes for existing adapter contracts
