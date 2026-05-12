# Delivery Adapters

## Purpose

Define the `delivery` adapter slot — separate from the `fulfillment` slot — and the `DeliveryAdapter` contract. Delivery adapters handle driver-based dispatch: in-house drivers, third-party logistics providers, or quote-only integrations.

Delivery is treated as a distinct concern from in-house fulfillment (pickup, dine-in, digital) and from carrier shipping (carrier-managed labels and tracking numbers). Each has its own adapter contract because the operational lifecycle differs.

The delivery pricing strategies that compute fees for these adapters live in [55-delivery-pricing.md](./55-delivery-pricing.md). The two are orthogonal — any adapter pairs with any strategy.

## Non-goals

- In-house fulfillment types (pickup, digital, dine-in, curbside) — covered by the existing `fulfillment` slot and [52-fulfillment-types.md](./52-fulfillment-types.md)
- Carrier shipping (Aramex, SMSA, DHL) — separate `shipping` slot, separate adapter contract
- Distance provider implementations (Google Maps, Mapbox) — see [55-delivery-pricing.md](./55-delivery-pricing.md)
- WhatsApp driver groups, push notifications to drivers — integration concerns layered as plugins on top, not adapter responsibilities

## Core decisions

- Delivery adapters live in their own top-level config slot: `delivery: [ ... ]`.
- The slot is an array of adapter factory calls; each factory's return type carries a literal `id` (with optional `id` override for multi-instance use).
- The `DeliveryAdapter` interface is purpose-built for driver dispatch. It does not share an interface with pickup, shipping, or carrier adapters.
- Adapters are dispatch-only. Fee calculation lives in delivery pricing strategies, not in the adapter.
- Multiple delivery adapters may be registered. Each delivery method picks one adapter via `adapter`.
- Adapters declare capabilities as **literal types**, not runtime booleans — TS refuses `commerce.delivery.cancel(...)` at compile time on adapters that don't support cancellation.
- Dispatch on order confirmation is **on by default**. Apps opt out via `orders: { autoDispatchDelivery: false }` and call `commerce.delivery.create()` manually if they need full control.

## Config slot

```ts
createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [moyasar({ secretKey: env.MOYASAR_SECRET })],

  delivery: [
    localDelivery(),
    leajlakDelivery({ apiKey: env.LEAJLAK }),
    parcelDelivery({ apiKey: env.PARCEL }),
    qmileDelivery({ apiKey: env.QMILE }),
  ],
})
```

The `delivery` slot is optional. When omitted (or empty), delivery-related features are dormant — no schema, no SDK surface, no HTTP routes.

Each factory's return type carries a literal `id`. TypeScript infers the union `'local' | 'leajlak' | 'parcel' | 'qmile'` from the tuple. The `deliveryMethod.adapter` field is type-checked against this union at compile time.

Pricing strategies are declared at app boot via the sibling `deliveryPricing: []` slot and passed inline to each method via typed factories — see [55-delivery-pricing.md](./55-delivery-pricing.md).

### Two confirmation flows

Merchants generally fall into one of two operational patterns:

- **Accept then dispatch (manual)** — Merchant reviews each order, accepts it, then explicitly dispatches the delivery. Common when the merchant wants a final check before paying a third-party provider.
- **Accept and auto-dispatch** — Merchant accepts the order and dispatch fires automatically. Common when delivery is in-house or the merchant trusts the chosen method blindly.

Both are first-class. The choice is governed by a hierarchical setting (app → merchant → method), and "manual" follows a **queue model** so admin UIs always have a row to attach buttons to.

### Auto-dispatch resolution

When `orders:confirmed` fires, core resolves auto-dispatch in this order:

```
deliveryMethod.autoDispatch !== 'inherit'   → use method setting
else merchant.autoDispatchDelivery !== null → use merchant setting
else app-level orders.autoDispatchDelivery  → use app default (true if unset)
```

App-level:

```ts
createCommerce({
  orders: {
    autoDispatchDelivery: false,    // app-wide default; defaults to true when omitted
  },
})
```

Merchant-level override (settable per merchant via `commerce.merchants.update`):

```ts
merchant.autoDispatchDelivery: boolean | null   // null means inherit from app
```

Per-method override (on the `deliveryMethod` row):

```ts
deliveryMethod.autoDispatch: 'inherit' | 'auto' | 'manual'   // default 'inherit'
```

### Queue model

Regardless of the resolved auto-dispatch decision, a `Delivery` row is **always** created on `orders:confirmed` in `pending` state with the order's `deliveryMethodId` set. The difference is only whether the adapter call fires:

- **auto** — core immediately calls `adapter.createDelivery(ctx)`. State moves to `dispatched`, `providerReference` is set.
- **manual** — the row sits in `pending`. Admin UIs list pending deliveries; a button calls `commerce.delivery.create({ orderId })` to dispatch (which transitions `pending` → `dispatched`).

Apps always have a row to attach UI / hooks / analytics to. The order's lifecycle and the delivery's lifecycle stay independent — the order is `confirmed` either way; the delivery is `pending` or `dispatched` depending on the flow.

### Default delivery method

For orders created without an explicit method selection (POS, phone-in, staff-created orders), core resolves a default at order-create time:

```
order.deliveryMethodId is supplied            → use it
else order has a branchId and the branch has  → branch.defaultDeliveryMethodId
  a default
else order has a merchantId and the merchant  → merchant.defaultDeliveryMethodId
  has a default
else order has a deliveryAddress              → throw CommerceValidationError('no_delivery_method')
else (no delivery needed)                     → deliveryMethodId stays null
```

Both default fields are settable via `commerce.merchants.update` and `commerce.branches.update`. Branch default takes precedence so a merchant operating in multiple cities can use Leajlak in Riyadh and QMile in Jeddah without conditional logic at the call site.

## `DeliveryAdapter` interface

```ts
interface DeliveryAdapter<C extends DeliveryCapabilities = DeliveryCapabilities> {
  capabilities: C

  createDelivery(ctx: CreateDeliveryContext): Promise<CreateDeliveryResult>
  cancelDelivery?: C['cancellation'] extends true
    ? (ctx: CancelDeliveryContext) => Promise<void>
    : never
  trackDelivery?: C['realtimeTracking'] extends true
    ? (ctx: TrackDeliveryContext) => Promise<DeliveryStatus>
    : never
  verifyWebhook?(payload: unknown, signature: string): Promise<DeliveryEvent>
}

interface DeliveryCapabilities {
  realtimeTracking: boolean      // provider supplies live driver location
  cancellation: boolean          // provider supports post-dispatch cancel
  proofOfDelivery: boolean       // signature/photo at handoff
  scheduling: boolean            // provider accepts scheduled dispatch windows
  estimatedTime: boolean         // provider returns ETA at dispatch
}

type CreateDeliveryContext = {
  order: Order
  deliveryAddress: Address
  branchAddress?: Address        // origin (present when tenancy.branches is on)
  method: DeliveryMethod
  request: RequestContext
}

type CreateDeliveryResult = {
  providerReference: string      // adapter's reference for this dispatch
  estimatedArrival?: Date
  metadata?: Record<string, unknown>
}
```

`createDelivery` is the only required method. Adapters declare capabilities as literal types — `capabilities: { cancellation: true, ... } as const` — and `cancelDelivery` / `trackDelivery` are only present in the type when the matching capability is `true`. This means TS rejects `commerce.delivery.cancel(...)` at compile time on adapters that don't support cancellation.

Example:

```ts
export function leajlakDelivery({ apiKey, id = 'leajlak' as const }: { apiKey: string; id?: string }) {
  return {
    id,
    capabilities: {
      cancellation: true,
      realtimeTracking: true,
      proofOfDelivery: true,
      scheduling: false,
      estimatedTime: true,
    } as const,
    createDelivery: async (ctx) => { /* ... */ },
    cancelDelivery: async (ctx) => { /* ... */ },
    trackDelivery: async (ctx) => { /* ... */ },
    verifyWebhook: async (payload, sig) => { /* ... */ },
  } satisfies DeliveryAdapter
}
```

## What the adapter does (and does not)

**Does:**
- Translate a `CreateDeliveryContext` into the provider's API call (POST to Leajlak, push to Redis queue for in-house drivers, etc.)
- Verify and parse provider webhooks into normalized `DeliveryEvent` shapes
- Report current delivery state when polled

**Does not:**
- Compute fees — that's the delivery pricing strategy's job
- Define delivery types — delivery is not a registry of types; the adapter IS the variation
- Notify drivers via WhatsApp / push / SMS — that's an application-level integration plugin layered on top, hooking the `orders:checkout:after` event or the delivery state machine

## `deliveryMethod` entity

When the `delivery` slot is active, core materializes the `deliveryMethod` table:

```ts
deliveryMethod = {
  id: string
  adapter: string                // references the delivery adapter registry (literal-typed)
  pricing: jsonb                 // tagged strategy config: { strategy: '<id>', ...settings }
  name: string
  enabled: boolean
  minOrderAmount: Money | null
  maxDistanceMeters: number | null
  autoDispatch: 'inherit' | 'auto' | 'manual'   // override the merchant/app default; 'inherit' is the default

  merchant: merchant().optional()  // hierarchical tenancy
  branch: branch().optional()      // per-branch pricing/methods

  metadata: jsonb
  createdAt: Date
  updatedAt: Date
}
```

The `pricing` column stores a tagged shape — `{ strategy: 'distance-based', ...typedSettings }` — produced by strategy factories. The factory is the input format for create/update; the persisted JSON carries the strategy tag so calculation can look up the right strategy later.

Each merchant configures their own delivery methods by picking an adapter and constructing pricing inline:

```ts
import { distanceBased } from '@commerce-kit/delivery-pricing-strategies'

await commerce.delivery.methods.create({
  adapter: 'leajlak',
  name: 'Riyadh Express',
  branch: 'br_riyadh',
  pricing: distanceBased({
    basePrice: money(1700, 'SAR'),
    baseDistanceMeters: 3000,
    perKm: money(100, 'SAR'),
  }),
  minOrderAmount: money(5000, 'SAR'),
  maxDistanceMeters: 20000,
})
```

The `pricing` argument is a discriminated union of strategy factory outputs. TS infers settings shape from the chosen factory — passing `flat({ amount })` settings to a method expecting `distanceBased({...})` is a compile error, not a runtime one.

Validation at write time:
- `adapter` must reference a registered delivery adapter
- `pricing.strategy` must reference a strategy registered in `deliveryPricing: []` (built-in or custom)
- Strategy settings are already type-checked at the factory call site, so runtime `validateSettings` becomes a defense-in-depth check rather than the primary gate

## Order fields when delivery is used

```ts
order = {
  ...existing fields,
  deliveryMethodId: string | null
  deliveryAddress: Address | null
  deliveryProviderReference: string | null    // set after createDelivery succeeds
}
```

The three fields are present (potentially null) when the `delivery` slot is active. They are absent from the schema entirely when delivery is dormant.

`deliveryAddress` is provided by the customer at checkout and persisted on the order. The pricing strategy reads it to compute the fee; the adapter reads it to dispatch.

## Lifecycle

The delivery lifecycle is independent of the order lifecycle. Core defines these delivery states:

```
pending → dispatched → in_transit → delivered
                    → cancelled
                    → failed
```

Each state transition is recorded in an append-only `deliveryTransition` log. State changes come from:
- Core operations (`commerce.delivery.cancel(...)`)
- Adapter webhooks (`verifyWebhook` returns a normalized event)
- Adapter polling (`trackDelivery` if implemented)

The order's main lifecycle (`placed → confirmed → fulfilled → completed`) does not directly mirror the delivery lifecycle — they evolve in parallel. Plugins can hook delivery state events to drive order transitions if a merchant wants (e.g., "auto-complete order when delivery is delivered").

## Server SDK

When `delivery` is active:

```ts
commerce.delivery.methods.{list, get, create, update, archive}
commerce.delivery.{create, cancel, get, list, track}
```

`commerce.delivery.create({ orderId })` is the explicit dispatch operation. It resolves the order's `deliveryMethodId`, calls `adapter.createDelivery(ctx)`, persists the provider reference, and emits the initial delivery transition.

**Auto-dispatch is the default.** When `orders.autoDispatchDelivery` is unset or `true`, core subscribes to `orders:confirmed` and runs `commerce.delivery.create` for any order with a non-null `deliveryMethodId`. Apps that need full control over dispatch timing set `autoDispatchDelivery: false` and call `commerce.delivery.create` themselves.

Admin tooling can always call `commerce.delivery.create` directly to re-dispatch a failed delivery regardless of the auto-dispatch setting.

Capability-gated operations are typed against the resolved adapter:

```ts
commerce.delivery.cancel({ id })   // OK if adapter.capabilities.cancellation === true
                                   // compile error if false
commerce.delivery.track({ id })    // OK if adapter.capabilities.realtimeTracking === true
```

## HTTP routes

When `delivery` is active:

```text
GET    /delivery/methods             POST   /delivery/methods
GET    /delivery/methods/:id         PATCH  /delivery/methods/:id
                                     DELETE /delivery/methods/:id

GET    /deliveries                   POST   /deliveries
GET    /deliveries/:id               POST   /deliveries/:id/cancel
GET    /deliveries/:id/track

POST   /webhooks/delivery/:adapterId
```

Webhook routes follow the same key convention as payment adapters — `/webhooks/delivery/leajlak`, `/webhooks/delivery/parcel`.

## Tenancy

`deliveryMethod` uses hierarchical tenancy via `merchant().optional()` + `branch().optional()`:

- A method with both columns null is platform-wide.
- A method with only `merchant` is merchant-wide (applies across all that merchant's branches).
- A method with both `merchant` and `branch` is branch-specific.

This matches Jaicome's per-branch pricing model (`location_provider_pricing`). A merchant operating in Riyadh and Jeddah can have two delivery methods using the same adapter (e.g., `leajlak`) with different per-branch pricing.

## Webhook event shape

Adapters that support webhooks return a normalized `DeliveryEvent`:

```ts
type DeliveryEvent =
  | { kind: 'dispatched'; providerReference: string; driverInfo?: DriverInfo }
  | { kind: 'in_transit'; providerReference: string; location?: Coordinates }
  | { kind: 'delivered'; providerReference: string; deliveredAt: Date; proof?: ProofOfDelivery }
  | { kind: 'cancelled'; providerReference: string; reason?: string }
  | { kind: 'failed'; providerReference: string; reason: string }
```

Core dispatches the event to the delivery state machine and to any plugin hooks subscribing to `delivery:*` events.

## Capability gating

Operations are gated by adapter capabilities at compile time:

- Capabilities are declared `as const` so `cancelDelivery` / `trackDelivery` only exist on the adapter type when the matching flag is `true`.
- `commerce.delivery.cancel(...)` resolves the adapter from the method and refuses the call at compile time when `cancellation` is `false`.
- `commerce.delivery.track(...)` similarly requires `realtimeTracking: true`. Adapters that surface state only via `verifyWebhook` don't expose `track` at all — callers consume the webhook events.

Runtime checks remain as defense-in-depth (in case an adapter is loaded dynamically), throwing `CommerceCapabilityError` if a capability is missing.

## Adapter implementations (illustrative, not normative)

These are the reference adapters expected to ship under `@commerce-kit/*`:

| Adapter | Provider | Capabilities |
|---|---|---|
| `localDelivery()` | in-house drivers | tracking via plugin layer, manual cancellation, no webhooks |
| `leajlakDelivery({ apiKey })` | Leajlak | real-time tracking, cancellation, webhooks |
| `parcelDelivery({ apiKey })` | Parcel | tracking, webhooks |
| `qmileDelivery({ apiKey })` | QMile | tracking, scheduling, webhooks |

Each ships as its own package with its own provider integration. Pricing math is shared via [55-delivery-pricing.md](./55-delivery-pricing.md) — adapters do not implement pricing.

## Cross-links

- Delivery pricing strategies: [55-delivery-pricing.md](./55-delivery-pricing.md)
- Tenancy and the `branch` entity: [12-tenancy.md](./12-tenancy.md)
- Adapter system overview and packaging: [50-adapter-system.md](./50-adapter-system.md)
- Calculation engine and the `core:delivery-fee` step: [35-calculation-engine.md](./35-calculation-engine.md)
- Type inference for `adapterId` / `pricingStrategyId` literal unions: [60-type-inference.md](./60-type-inference.md)

## Future RFCs

- Multi-leg delivery (warehouse → hub → customer)
- Driver assignment within in-house local delivery (separate from external dispatch)
- Proof-of-delivery storage and retrieval
- Scheduled delivery windows surfaced in checkout
