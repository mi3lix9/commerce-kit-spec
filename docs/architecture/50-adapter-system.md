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

All adapter slots use **arrays**. Each adapter factory returns a value whose `id` is a string literal (or an `id` override the caller supplies). TypeScript infers the union of registered IDs from the tuple, so downstream fields like `order.payment.adapter` are typed as the literal union of registered payment adapter IDs.

```ts
export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),

  payments: [
    moyasar({ secretKey: env.MOYASAR_SECRET }),
    tabby({ apiKey: env.TABBY_API_KEY }),
  ],

  fulfillments: [
    pickupFulfillment(),
    digitalFulfillment(),
  ],

  deliveries: [
    localDelivery({ id: 'riyadh' }),
    localDelivery({ id: 'jeddah' }),
  ],

  storage: [r2({ bucket: env.R2_BUCKET, accountId: env.CF_ACCOUNT_ID })],

  payout: [hyperpay({ apiKey: env.HYPERPAY_API_KEY })],

  scheduler: bullmq({ redis: env.REDIS_URL }),

  plugins: [coupons(), marketplace()],
})
```

`database` and `scheduler` are single adapters (not arrays) because only one of each is active at a time. All other optional adapter slots accept arrays and are dormant when omitted or empty.

Each factory may accept an optional `id` override to disambiguate multiple instances of the same provider — `localDelivery({ id: 'riyadh' })` vs `localDelivery({ id: 'jeddah' })`. Without an override, the factory's default `id` is used. Duplicate IDs across a slot are rejected at startup.

## Packaging model

### Bundled in `commerce-kit`

- `drizzleAdapter`
- adapter interfaces for `payment`, `fulfillment`, `storage`, and `payout`
- `plugin` factory, `table` and column helpers (`text`, `integer`, `merchant`, `branch`, etc.)
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
  verifyWebhook?(payload: unknown, signature: string): Promise<WebhookEvent>
  chargeOffSession?(customerId: string, amount: number, currency: string): Promise<OffSessionResult>
}
```

`verifyWebhook` is optional because synchronous adapters (e.g., COD) don't have an inbound webhook stream.

Not every provider supports every payment flow; capabilities are explicit.

```ts
interface PaymentCapabilities {
  flow: 'synchronous' | 'redirect' | 'inline'   // see "Payment flow" below
  supportsCapture: boolean                      // can capture an authorized payment later
  supportsVoid: boolean                         // can void an authorization without a refund
  supportsRefund: boolean                       // can refund a captured/paid payment
  supportsPartialRefund: boolean
  supportsOffSession: boolean                   // can charge a saved customer without a session
  voidableStates?: Array<'AUTHORIZED' | 'CAPTURED'>   // transaction types eligible for cancel(); default ['AUTHORIZED']
}
```

### Payment flow

Adapters declare how `authorize()` completes:

| `flow` | After `authorize()` | Example providers |
|---|---|---|
| `'synchronous'` | Payment is in a deterministic state with no async confirmation expected. No webhook. Manual transition to `paid` via `orders.setPaid`. | Cash on delivery, manual bank transfer, in-store payment |
| `'redirect'` | Customer must follow `paymentUrl` to complete payment at the provider's hosted page. Confirmation arrives via webhook. | Moyasar invoice, PayPal, hosted-page providers |
| `'inline'` | Customer completes payment in-page via the provider's JS SDK using `inlineSecret`. Confirmation arrives via webhook. | Stripe Elements, Square Card |

Adapter methods return a discriminated **outcome** rather than a status string. Core inspects the outcome and decides whether to append a `paymentTransaction` row, transition `order.status`, or return the in-flight handle to the caller without persisting anything.

```ts
type AuthorizeResult =
  | { outcome: 'authorized';      providerReference: string; amount: number }
  | { outcome: 'captured';        providerReference: string; amount: number }
  | { outcome: 'requires_action'; providerReference: string; paymentUrl?: string; inlineSecret?: string }
  | { outcome: 'failed';          providerReference?: string; reason: string }

type CaptureResult =
  | { outcome: 'captured'; providerReference: string; amount: number }
  | { outcome: 'failed';   reason: string }

type RefundResult =
  | { outcome: 'refunded'; providerReference: string; amount: number }
  | { outcome: 'failed';   reason: string }

type CancelResult =
  | { outcome: 'voided'; providerReference: string }
  | { outcome: 'failed'; reason: string }
```

How outcomes are persisted:

| Outcome | Core action |
|---|---|
| `authorized` | Append `paymentTransaction { type: 'AUTHORIZE', amount, providerReference }`. |
| `captured` | Append `paymentTransaction { type: 'CAPTURE', amount, providerReference }`. |
| `refunded` | Append `paymentTransaction { type: 'REFUND', amount, providerReference }`. |
| `voided` | Append `paymentTransaction { type: 'VOID', amount: 0, providerReference }`. |
| `requires_action` | **Nothing persisted.** `paymentUrl` / `inlineSecret` are returned synchronously to the caller and may be cached by the application in transient storage (e.g. Redis). The order stays in `placed`. |
| `failed` | **No transaction row.** `order.status` transitions to `failed` (per the order state machine in [20-data-model.md](./20-data-model.md)). |

For `flow: 'synchronous'`:
- `authorize()` returns `outcome: 'authorized'` (implicit hold by the offline gateway) or `outcome: 'captured'` (e.g. cash collected before the call).
- The order is in `placed` state, awaiting manual confirmation.
- `commerce.orders.setPaid({ id })` is the path that collects payment for synchronous flows; it calls `adapter.capture(...)` under the hood and the resulting `captured` outcome appends a `CAPTURE` row.
- `setPaid` validates that the adapter declares `flow: 'synchronous'`. Calling it on a `redirect` or `inline` adapter throws — those flows must be confirmed via webhook.

For `flow: 'redirect'` and `'inline'`:
- `verifyWebhook` is required.
- `orders.setPaid` is not a valid path; payment confirmation must arrive via the verified webhook.

Capabilities drive `orders.refund` orchestration:

- When `orders.refund` runs (mode `'auto'`), it inspects `latest(paymentTransaction)` for the order.
- If the latest transaction's `type` is in `voidableStates` (default `['AUTHORIZED']`), the operation calls `adapter.cancel(providerReference)` first.
- On `failed` outcome from `cancel()`, or when the latest type is not voidable, the operation calls `adapter.refund(providerReference, amount, reason)`.
- An adapter that auto-captures or does not distinguish void from refund declares `supportsVoid: false`; `orders.refund` then always refunds.

Each successful capture, refund, or void appends a new `paymentTransaction` row. Existing rows are never mutated. There is no "primary payment row" to keep up to date.

### Payment status is derived, not stored

Commerce Kit does not persist a `paymentStatus` column anywhere. "Current payment state" is a query over the `paymentTransaction` ledger plus `order.status`:

```text
latest = latest(paymentTransaction WHERE orderId = ?) by seq

if latest exists:
  AUTHORIZE  → AUTHORIZED
  CAPTURE    → CAPTURED            (or PARTIALLY_REFUNDED / REFUNDED once REFUND rows accumulate)
  REFUND     → PARTIALLY_REFUNDED  if Σrefund < Σcapture, else REFUNDED
  VOID       → VOIDED

else if order.paymentAdapterId is set:
  order.status = placed   → INITIATED or REQUIRES_ACTION (customer-action in flight)
  order.status = failed   → FAILED (declined / abandoned / expired)
  order.status = cancelled→ VOIDED (pre-authorization cancellation)

else:
  no payment context yet
```

The keywords `AUTHORIZED`, `CAPTURED`, `PARTIALLY_REFUNDED`, `REFUNDED`, `VOIDED`, `FAILED`, `INITIATED`, `REQUIRES_ACTION` are the **derived view** vocabulary returned by `commerce.orders.paymentState(orderId)` and surfaced in audit logs. They are not stored as strings on any row.

Adapters never declare a status. They return an outcome (`'authorized' | 'captured' | 'refunded' | 'voided' | 'requires_action' | 'failed'`), and core decides what to persist. This keeps the adapter contract small and the data model immutable in the strict sense — every persisted row is a money-moved event.

#### Transaction type vs. derived status

The `paymentTransaction` ledger carries only `type`:

```ts
type PaymentTransactionType = 'AUTHORIZE' | 'CAPTURE' | 'REFUND' | 'VOID'
```

A row exists because the operation settled. There is no `status` field — the row's existence is the success signal. See [20-data-model.md](./20-data-model.md).

The derived status union (`AUTHORIZED | CAPTURED | …`) lives only in the SDK response shape and in log events; it is never serialized to a column.

#### Transition rules

Because the ledger is append-only and the derived view is recomputed from it, "transitions" exist only as the allowed progression of transaction types per order:

```text
∅ ──► AUTHORIZE ──► CAPTURE ──► REFUND (one or more, ≤ Σcaptures)
  ├──► CAPTURE                  (auto-capture / 3-D Secure complete)
  └──► VOID                     (cancel before capture)
```

Each `paymentTransaction` insert is validated against `latest(paymentTransaction)`:

- `AUTHORIZE` requires no prior row.
- `CAPTURE` requires either no prior row (auto-capture) or a prior `AUTHORIZE`.
- `REFUND` requires `Σcapture > Σrefund` after this insert.
- `VOID` requires the latest row to be `AUTHORIZE`.

A webhook event that would violate this graph (e.g. `REFUND` after `VOID`) is rejected and the adapter raises `CommerceProviderError`. See [15-errors.md](./15-errors.md).

#### Capability gates

- `capabilities.voidableStates` is typed `Array<'AUTHORIZED' | 'CAPTURED'>` and defaults to `['AUTHORIZED']`. It governs which latest-transaction types make `cancel()` an eligible refund path. An adapter that can void a captured payment within a settlement window declares `['AUTHORIZED', 'CAPTURED']`.
- `orders.refund` reads `latest(paymentTransaction)` to decide void-vs-refund.
- `orders.setPaid` is valid only when `capabilities.flow === 'synchronous'` AND the latest transaction is missing or `AUTHORIZE`. It calls `adapter.capture(...)` and persists the resulting `CAPTURE` row.

#### Mapping provider outcomes

Every adapter ships a small `mapOutcome(providerPayload) → AuthorizeResult['outcome']` (and equivalent for capture/refund/cancel). Unknown provider statuses **must throw**, not default to `'failed'` — silently failing downstream automation is worse than a loud error. See [`docs/examples/moyasar-adapter.md`](../examples/moyasar-adapter.md) for a worked mapping.

### Multiple payment adapters

```ts
createCommerce({
  payments: [
    moyasar({ secretKey: env.MOYASAR_SECRET }),
    tabby({ apiKey: env.TABBY_API_KEY }),
  ],
})
```

Each factory's return type carries a literal `id`. TypeScript infers the union `'moyasar' | 'tabby'` from the tuple, so passing an unknown `adapterId` at checkout is a compile error.

At checkout, the caller selects the adapter by ID:

```ts
await commerce.orders.checkout({
  items: cart.items,
  payment: { adapterId: 'moyasar' },
})
```

Webhook URLs follow the same convention: `/webhooks/payment/moyasar`, `/webhooks/payment/tabby`.

### Adapter-id uniqueness

Adapter `id` literals must be unique within a slot. The same factory used twice (e.g. two `localDelivery` instances for different regions) must supply distinct `id` overrides. Duplicates are rejected at the type level — TypeScript narrows the slot tuple and reports the collision at the `createCommerce()` call site, not at runtime:

```ts
createCommerce({
  deliveries: [
    localDelivery({ id: 'riyadh' }),
    localDelivery({ id: 'riyadh' }),   // ❌ Type error: duplicate adapter id 'riyadh' in slot 'deliveries'
  ],
})
```

The same uniqueness rule applies to `payments`, `fulfillments`, `payouts`, and `storage`. The `scheduler` slot is single-valued and does not need a tuple-uniqueness check.

### Capability gating on the SDK type

Adapter capabilities are declared as literal types (`capabilities: { supportsVoid: true } as const`), and the SDK surface is narrowed against the **union** of capabilities across all registered adapters in the slot. A method exists on the typed SDK only if at least one configured adapter declares the matching capability. If no adapter does, the method is absent — calling it is a compile error, not a runtime `CommerceCapabilityError`.

```ts
const m = moyasar({ secretKey: env.MOYASAR_SECRET })   // supportsVoid: false  as const
const h = hyperpay({ apiKey: env.HYPERPAY_KEY })       // supportsVoid: true   as const

// Slot with only moyasar
createCommerce({ payments: [m] })
await commerce.payments.void({ id })            // ❌ Property 'void' does not exist

// Slot with both — `void` exists, but adapterId narrows further
createCommerce({ payments: [m, h] })
await commerce.payments.void({ id, adapterId: 'hyperpay' })   // ✅
await commerce.payments.void({ id, adapterId: 'moyasar' })    // ❌ moyasar.capabilities.supportsVoid is false
```

The same union-and-narrow pattern applies to every slot: delivery (`cancellation`, `realtimeTracking`, `proofOfDelivery`), payout (partial release, reversal), fulfillment (label printing, return labels), and so on. Each adapter's capabilities are visible on `commerce.metadata.adapters()` for runtime introspection.

> **Implementation note.** The current runtime allows a single active adapter per slot to be selected at call time; the multi-adapter typed surface above is the v1 target shape. Code authored against the typed surface keeps working as multi-adapter dispatch lands — the typed contract is intentionally the larger of the two.

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

Fulfillment is optional and dormant until configured via `createCommerce({ fulfillments: [ ... ] })`.

Fulfillment is the public abstraction for carrier shipping, local delivery, pickup, and digital delivery. The public API is unified even when provider adapters have different capabilities.

```ts
interface FulfillmentAdapter {
  types: string[]                // type IDs from the fulfillment type registry
  capabilities: FulfillmentCapabilities

  listMethods?(ctx: ListFulfillmentMethodsContext): Promise<FulfillmentMethodCandidate[]>
  createFulfillment?(ctx: CreateFulfillmentContext): Promise<CreateFulfillmentResult>
  cancelFulfillment?(ctx: CancelFulfillmentContext): Promise<CancelFulfillmentResult>
  track?(ctx: TrackFulfillmentContext): Promise<FulfillmentTracking>
  createLabel?(ctx: CreateLabelContext): Promise<FulfillmentLabel>
  verifyWebhook?(payload: unknown, signature: string): Promise<FulfillmentWebhookEvent>
}

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

`types` is an array of type IDs from the fulfillment type registry. Core pre-registers `'shipping'`, `'delivery'`, `'pickup'`, `'digital'`; plugins or inline config contribute additional types like `'restaurant:dinein'`. The adapter declares which subset it handles. See [52-fulfillment-types.md](./52-fulfillment-types.md) for registration, schemas, and validation.

Adapters consume `order.fulfillmentTypeData` (the typed payload validated against the registered schema) directly inside their handlers. They do not register types — they only consume them.

Config uses an array. The factory's return type carries the `id` as a literal. Multiple instances of the same adapter factory pass an explicit `id` override:

```ts
createCommerce({
  fulfillments: [
    shippo({ apiKey: env.SHIPPO_API_KEY }),
    localDelivery({ id: 'riyadh' }),
    localDelivery({ id: 'jeddah' }),
    storePickup(),
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
  storage: [r2({ bucket: env.R2_BUCKET, accountId: env.CF_ACCOUNT_ID })],
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
  payout: [hyperpay({ apiKey: env.HYPERPAY_API_KEY })],
})
```

Normative rules:

- Payout activates only when a payout provider is registered.
- Payout activation is commonly required by applications using marketplace mode (`tenancy.checkout: 'split'`) for per-merchant settlement, but activation itself is still owned by core.
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
  scheduler: bullmq({ redis: env.REDIS_URL }),
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
