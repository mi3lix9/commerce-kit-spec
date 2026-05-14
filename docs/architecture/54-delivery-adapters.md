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

- Delivery adapters live in their own top-level config slot: `deliveries: [ ... ]`.
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

  deliveries: [
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

The contract is designed so the adapter does one thing: translate between Commerce Kit's normalized shapes and the provider's wire format. Inputs arrive already normalized (the package extracts and computes them from the order, address book, payment, and method config); outputs are normalized too (the framework persists the same shape regardless of provider).

```ts
interface DeliveryAdapter<C extends DeliveryCapabilities = DeliveryCapabilities> {
  capabilities: C

  createDelivery(ctx: CreateDeliveryContext): Promise<CreateDeliveryResult>

  cancelDelivery?: C['cancellation'] extends true
    ? (ctx: CancelDeliveryContext) => Promise<CancelDeliveryResult>
    : never

  trackDelivery?: C['realtimeTracking'] extends true
    ? (ctx: TrackDeliveryContext) => Promise<TrackDeliveryResult>
    : never

  quoteDelivery?: C['quotation'] extends true
    ? (ctx: QuoteDeliveryContext) => Promise<QuoteDeliveryResult>
    : never

  listDrivers?: C['driverSelection'] extends true
    ? (ctx: ListDriversContext) => Promise<DriverInfo[]>
    : never

  getAirwayBill?: C['airwayBill'] extends true
    ? (ctx: GetAirwayBillContext) => Promise<AirwayBill>
    : never

  webhook?: DeliveryWebhookConfig
}

interface DeliveryCapabilities {
  realtimeTracking: boolean      // provider supplies live driver location
  cancellation: boolean          // provider supports post-dispatch cancel
  proofOfDelivery: boolean       // signature/photo at handoff
  scheduling: boolean            // provider accepts scheduled dispatch windows
  estimatedTime: boolean         // provider returns ETA at dispatch
  cashOnDelivery: boolean        // provider collects cash from recipient on handoff
  multiStop: boolean             // provider accepts multiple delivery stops in one task
  vehicleSelection: boolean      // provider accepts a vehicle type hint at dispatch
  airwayBill: boolean            // provider returns a printable AWB / shipping label
  driverContact: boolean         // provider exposes driver name/phone (independent of live tracking)
  driverSelection: boolean       // merchant can pick a specific driver at dispatch
  quotation: boolean             // provider returns a binding fee quote pre-dispatch
  returnFlow: boolean            // provider supports return-to-sender lifecycle ('returned' state)
}
```

### Inputs — what the framework hands the adapter

Inputs are pre-computed by Commerce Kit. The adapter never reaches into `order.payment`, `order.metadata`, or merchant settings to derive a value — if it needs something, the framework puts it on the context.

```ts
type CreateDeliveryContext = {
  order: NormalizedOrder         // id, currency, totals, notes, payment.method, fulfillmentTypeData
  recipient: Recipient           // pre-extracted from order.customer + delivery address
  deliveryAddress: Address       // structured + formatted single-line
  branchAddress?: Address        // pickup origin (when tenancy.branches is on)
  method: DeliveryMethod         // the resolved deliveryMethod row, incl. metadata
  cashOnDelivery?: Money         // present when capabilities.cashOnDelivery AND order.payment.method === 'cod'
  scheduledPickupAt?: Date       // present when capabilities.scheduling AND the order requested a window
  vehicleHint?: VehicleType      // present when capabilities.vehicleSelection AND a hint was supplied
  preferredDriverId?: string     // present when capabilities.driverSelection AND a driver was picked
  stops?: DeliveryStop[]         // present when capabilities.multiStop AND the order is multi-drop
  request: RequestContext        // idempotencyKey, locale, requestedAt
}

type CancelDeliveryContext = {
  providerReference: string
  reasonCode?: CancellationCode         // structured code for providers that require one
  reason?: string                       // free-text detail; falls back to a default when the provider needs something
  method: DeliveryMethod
  request: RequestContext
}

type CancellationCode =
  | 'merchant_request'         // merchant changed their mind
  | 'customer_request'         // customer asked to cancel
  | 'address_invalid'          // dispatched against a bad address
  | 'duplicate'                // cancelling a double-dispatch
  | 'fraud_suspected'
  | 'other'

type TrackDeliveryContext = {
  providerReference: string
  method: DeliveryMethod
  request: RequestContext
}

type QuoteDeliveryContext = {
  // Same shape as CreateDeliveryContext minus the order reference — quotes are
  // generated pre-dispatch, often before an order id even exists. The framework
  // synthesizes a transient context from the cart and the resolved method.
  recipient: Recipient
  deliveryAddress: Address
  branchAddress?: Address
  method: DeliveryMethod
  scheduledPickupAt?: Date
  vehicleHint?: VehicleType
  stops?: DeliveryStop[]
  request: RequestContext
}

type ListDriversContext = {
  method: DeliveryMethod
  branchAddress?: Address        // for proximity-aware listings
  request: RequestContext
}

type GetAirwayBillContext = {
  providerReference: string
  method: DeliveryMethod
  format?: 'pdf' | 'png'         // hint; provider may ignore
  request: RequestContext
}

type DeliveryStop = {
  id: string                     // Commerce Kit-side stop id (stable across re-dispatch)
  recipient: Recipient
  address: Address
  cashOnDelivery?: Money         // per-stop COD when capabilities.cashOnDelivery is true
  notes?: string
}

type Recipient = {
  name: string
  phone: string
  email?: string
}

type Address = {
  formatted: string              // single-line display ("123 King Fahd Rd, Riyadh 12345")
  components?: AddressComponents // structured fields; see below
  coordinates?: Coordinates      // always present for delivery addresses; optional for branch
  notes?: string                 // gate code, floor, "ring buzzer twice"
}

type AddressComponents = {
  line1: string                  // primary street line
  line2?: string                 // apartment, unit, suite
  district?: string              // neighbourhood / 'حي' in KSA, 'منطقة' in BH
  city: string
  region?: string                // province / state / emirate
  postalCode?: string
  country: string                // ISO 3166-1 alpha-2 ('SA', 'BH', 'AE')
}

type Coordinates = { lat: number; lng: number }

type VehicleType = 'bike' | 'motorcycle' | 'car' | 'van' | 'truck'
```

### Outputs — what the framework expects back

Every method returns a normalized result. The provider's raw payload always travels on `raw` so plugins and admin UIs can inspect it; `metadata` is for adapter-specific extras that don't fit the normalized shape but are not the raw payload itself.

```ts
type CreateDeliveryResult = {
  providerReference: string             // operational ref for downstream calls back into the provider
  customerTrackingNumber?: string       // customer-facing tracking string ("QM-2026-100123"); shown on order page / SMS
  state: DeliveryState                  // typically 'dispatched'; 'failed' allowed when provider declines at create time
  estimatedArrival?: Date
  fee?: Money                           // present when the provider returns a per-dispatch fee
  driver?: DriverInfo                   // present when capabilities.driverContact AND a driver is assigned at create time
  airwayBill?: AirwayBill               // present when capabilities.airwayBill is true
  reason?: string                       // human-readable explanation when state === 'failed'
  failureReason?: FailureReason         // structured classification when state === 'failed'
  metadata?: Record<string, unknown>
  raw: unknown                          // the untouched provider payload
}

type FailureReason = {
  code:
    | 'recipient_unavailable'    // doorbell unanswered, recipient phone unreachable
    | 'address_invalid'          // bad lat/lng or unreachable address
    | 'refused'                  // recipient refused the package
    | 'no_drivers'               // provider has no available drivers
    | 'rate_limited'             // provider throttled the dispatch
    | 'provider_error'           // 5xx, network, malformed response
    | 'unknown'                  // adapter couldn't classify
  message: string                // provider-supplied human-readable detail
}

type QuoteDeliveryResult = {
  fee: Money                     // binding fee the provider will charge if the same payload is dispatched
  estimatedArrival?: Date
  validUntil?: Date              // quote expiry; absent means single-use
  vehicleType?: VehicleType      // the vehicle the quote was priced against
  quoteId?: string               // pass to createDelivery's method.metadata to lock the quote
  raw: unknown
}

type CancelDeliveryResult = {
  providerReference: string
  state: 'cancelled' | 'failed'         // 'failed' when provider rejects (e.g., post-pickup)
  reason?: string
  failureReason?: FailureReason         // structured classification when state === 'failed'
  raw: unknown
}

type TrackDeliveryResult = {
  providerReference: string
  customerTrackingNumber?: string
  state: DeliveryState
  estimatedArrival?: Date
  currentLocation?: Coordinates         // present when capabilities.realtimeTracking is true AND a driver is moving
  driver?: DriverInfo
  proof?: ProofOfDelivery               // present when state === 'delivered' AND capabilities.proofOfDelivery is true
  history?: DeliveryTransition[]        // ordered audit trail if the provider returns one
  failureReason?: FailureReason         // present when state === 'failed'
  metadata?: Record<string, unknown>
  raw: unknown
}

type DriverInfo = {
  id: string                     // provider-side driver id; round-trips as ctx.preferredDriverId on createDelivery
  name: string
  phone?: string
  photoUrl?: string
  vehicle?: { type?: VehicleType; plate?: string }
  currentLocation?: Coordinates  // last known location at the time of the read
  currentLoad?: number           // active orders / queue depth, when the provider exposes it
}

type AirwayBill = {
  format: 'pdf' | 'png'
  url?: string                   // when the provider serves it via a signed URL
  data?: Uint8Array              // when the provider returns the file body inline
}

type ProofOfDelivery = {
  signedAt: Date
  signatureUrl?: string
  photoUrl?: string
  recipientName?: string
  notes?: string
}

type DeliveryTransition = {
  state: DeliveryState
  at: Date
  note?: string
}
```

### Webhook contract

The webhook adapter returns a normalized `DeliveryEvent` — same shape regardless of provider. Plugins and the state machine consume `DeliveryEvent`, not the raw payload.

```ts
interface DeliveryWebhookConfig<TPayload = unknown, TOptions = unknown> {
  signatureHeader?: string                            // default header name the framework reads from request.headers
                                                      // (e.g., 'x-parcel-signature'). Merchants override per tenant via
                                                      // options if their account is configured with a different name.
  verify(input: { rawBody: string; headers: Record<string, string>; options: TOptions }): boolean
  extract(rawBody: string): TPayload
  providerReference(payload: TPayload): string
  toEvent(payload: TPayload): DeliveryEvent           // <- the normalization seam
  deliveryKey?(payload: TPayload): string             // webhook idempotency key
}

type DeliveryEvent =
  | { kind: 'dispatched';  providerReference: string; estimatedArrival?: Date; raw: unknown }
  | { kind: 'accepted';    providerReference: string; driver?: DriverInfo; raw: unknown }      // proposed (see states list)
  | { kind: 'picked_up';   providerReference: string; pickedUpAt: Date; raw: unknown }         // proposed
  | { kind: 'in_transit';  providerReference: string; currentLocation?: Coordinates; driver?: DriverInfo; raw: unknown }
  | { kind: 'arrived';     providerReference: string; arrivedAt: Date; raw: unknown }          // proposed
  | { kind: 'delivered';   providerReference: string; deliveredAt: Date; proof?: ProofOfDelivery; raw: unknown }
  | { kind: 'returned';    providerReference: string; returnedAt: Date; reason?: string; raw: unknown }  // proposed
  | { kind: 'cancelled';   providerReference: string; reason?: string; raw: unknown }
  | { kind: 'failed';      providerReference: string; reason: string; raw: unknown }
```

### What the framework does with the normalized output

| Field on result | What core does |
|---|---|
| `providerReference` | Persisted on `Delivery.providerReference`; the lookup key for every subsequent call back to this dispatch. |
| `customerTrackingNumber` | Persisted on `Delivery.customerTrackingNumber`; surfaced on `commerce.delivery.get(id)` and customer-facing order pages. |
| `state` | Transitions the `Delivery` row to that state; appended to the `deliveryTransition` log. |
| `estimatedArrival` | Persisted on `Delivery.estimatedArrival`. |
| `failureReason` | Persisted on the transition row; emitted on the `delivery:failed` event so plugins can branch on `failureReason.code` (e.g., retry on `no_drivers`, escalate on `address_invalid`). |
| `driver`, `currentLocation` | Persisted on `Delivery.metadata.driver` / `.currentLocation`; exposed via `commerce.delivery.get(...)` in the normalized shape. |
| `proof` | Persisted on `Delivery.metadata.proof`; only set on `delivered` transitions. |
| `fee` | Persisted on `Delivery.metadata.providerFee`; useful when the merchant wants to reconcile the provider's billing against the strategy-computed fee on the order. |
| `airwayBill` | Persisted (URL or stored blob via the storage adapter); exposed via `commerce.delivery.airwayBill(id)` when the capability is on. |
| `reason` | Stored on the transition row, surfaced in error responses, included in the `deliveryFailed` event. |
| `metadata` | Merged into `Delivery.metadata` (adapter extras the normalized shape doesn't model). |
| `raw` | Stored on the transition row's `raw` column for forensics; never read by core logic. |

Because outputs are normalized, `commerce.delivery.get(id)` returns the same shape regardless of which adapter dispatched — plugins, admin UIs, and customer-facing tracking pages are provider-agnostic.

### Persisted `Delivery` entity

The row materialized in the `delivery` table when the slot is active. A row is created in `pending` state on every confirmed order with a `deliveryMethodId`; the adapter call upgrades it to `dispatched` (or `failed`):

```ts
type Delivery = {
  id: string
  orderId: string
  methodId: string
  adapterId: string                                       // resolved delivery adapter id
  providerReference: string | null                        // operational ref; null until the adapter dispatches
  customerTrackingNumber: string | null                   // customer-facing tracking string; null when the provider doesn't expose one
  state: DeliveryState
  estimatedArrival: Date | null
  metadata: Record<string, unknown> | null                // adapter-supplied payload
  merchant: MerchantId | null                             // present when tenancy.merchants is on
  branch: BranchId | null                                 // present when tenancy.branches is on
  createdAt: Date
  updatedAt: Date
}
```

`DeliveryState` is the union exposed by the framework. Not every transition is mandatory — providers that don't emit a given state are simply absent from it (e.g., a provider that goes `dispatched → in_transit → delivered` skips `accepted`, `picked_up`, and `arrived`). Forward-only jumps are legal:

```ts
type DeliveryState =
  | 'pending'        // row created, adapter not yet called (manual dispatch queue)
  | 'dispatched'     // adapter accepted the request; no driver yet
  | 'accepted'       // driver assigned and acknowledged the dispatch
  | 'picked_up'      // package is in the driver's hands; cancellation often blocked from here
  | 'in_transit'     // driver is moving (toward pickup or toward dropoff)
  | 'arrived'        // driver at destination, awaiting handoff
  | 'delivered'      // terminal: handoff complete
  | 'returned'       // terminal: returned to sender (only when capabilities.returnFlow is true)
  | 'cancelled'      // terminal: cancelled before delivery
  | 'failed'         // terminal: provider rejected or delivery could not be completed
```

State transitions flow through the `DeliveryEvent` stream (see below); apps should not mutate `delivery` rows directly.

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

### Authoring with `createDeliveryAdapter`

The raw `DeliveryAdapter` interface is the contract; `createDeliveryAdapter` is the recommended way to author one. The helper is the delivery-slot sibling of [`createPaymentAdapter`](./50-adapter-system.md#authoring-with-createpaymentadapter) and owns the parts every adapter would otherwise repeat — literal-id inference, options validation, capability typing, a configured `fetch`, normalized state mapping, and the webhook flow shell — leaving the developer to write provider-specific logic: a configured fetch, a state mapper, and per-method request bodies.

```ts
import { createDeliveryAdapter } from "commerce-kit"
import * as v from "valibot"

export const myProvider = createDeliveryAdapter({
  id: "my-provider",                                          // default literal id; overridable at call site
  options: v.object({ apiKey: v.string() }),                  // Standard Schema; validated at createCommerce()

  capabilities: {
    cancellation: true,
    realtimeTracking: false,
    proofOfDelivery: true,
    scheduling: true,
    estimatedTime: true,
  },

  fetch: ({ options }) => (input, init) =>                    // returns a standard Response-returning fetch
    globalThis.fetch(`https://api.example.com${input}`, {
      ...init,
      headers: { ...init?.headers, Authorization: `Bearer ${options.apiKey}` },
    }),

  createDelivery: async ({ fetch, ctx }) => {
    const payload = await fetch("/dispatch", { method: "POST", body: JSON.stringify({ /* … */ }) }).then(r => r.json())
    return {
      providerReference: payload.id,
      state: "dispatched",
      estimatedArrival: payload.eta ? new Date(payload.eta) : undefined,
      raw: payload,
    } satisfies CreateDeliveryResult
  },

  cancelDelivery: async ({ fetch, providerReference }) => {
    const payload = await fetch(`/dispatch/${providerReference}`, { method: "DELETE" }).then(r => r.json())
    return { providerReference, state: "cancelled", raw: payload }
  },

  trackDelivery: async ({ fetch, providerReference }) => {
    const payload = await fetch(`/dispatch/${providerReference}`).then(r => r.json())
    return {
      providerReference,
      state: toState(payload.status),                         // local function the developer writes
      currentLocation: payload.driver?.location,
      driver: payload.driver && { name: payload.driver.name, phone: payload.driver.phone },
      raw: payload,
    }
  },

  webhook: {
    verify: ({ rawBody, headers, options }) => /* boolean */,
    extract: (raw) => JSON.parse(raw),
    providerReference: (p) => p.id,
    toEvent: (p) => ({
      kind: toKind(p.status),                                 // local function: "dispatched" | "in_transit" | …
      providerReference: p.id,
      driver: p.driver && { name: p.driver.name, phone: p.driver.phone },
      raw: p,
    }),
    deliveryKey: (p) => `${p.id}:${p.status}:${p.updated_at}`,
  },
})
```

Normative rules for the helper (these mirror `createPaymentAdapter` — cross-reference rather than restate):

- **`id` lives on the factory**, not in `options`. Override at the call site for multi-instance setups: `myProvider({ id: "my-provider-eu", ... })`.
- **`fetch` is standard Web fetch.** The factory returns a `(input, init) => Promise<Response>` closure. Authoring helpers (token caching, request signing, region routing) compose at the closure boundary, not via invented sugar methods.
- **Capabilities are literal-typed.** Pass the object inline — the helper applies `as const` for you so `cancelDelivery` / `trackDelivery` are typed only when the matching capability is `true`. Declaring `cancellation: false` and shipping a `cancelDelivery` method is a compile error.
- **The adapter owns the mapping.** Each method body translates the provider payload into the normalized output shape (`CreateDeliveryResult`, `TrackDeliveryResult`, `CancelDeliveryResult`, `DeliveryEvent`). The framework does not run a generic "status mapper" — there is no `mapState` config field. State derivation is a normal local function the developer writes and reuses across methods and `webhook.toEvent`.
- **`raw` is required on every output.** The full provider payload travels on `raw` so plugins, admin UIs, and forensics can inspect it; core logic never reads it.
- **Inputs are pre-computed.** `ctx.recipient.name`, `ctx.recipient.phone`, `ctx.cashOnDelivery`, `ctx.scheduledPickupAt`, `ctx.vehicleHint`, and `ctx.deliveryAddress` arrive already normalized. The method body never reaches into `ctx.order.payment` or `ctx.order.metadata` to derive these — if a needed value is missing, the framework's normalization layer is the bug, not the adapter.
- **Methods may short-circuit with `state: 'failed'`** for cases where the provider returns `200` but rejects the dispatch in the payload body (analogous to Tabby's pre-scoring rejection on the payment side).
- **`webhook.verify` returns a boolean.** HMAC providers `timingSafeEqual` on digests; header-token providers do a constant-time string compare. The framework rejects with `CommerceWebhookError` on `false`. The verified payload flows through `extract` → `toEvent` to produce a `DeliveryEvent`.
- **Method throws → `state: 'failed'`.** Any `CommerceProviderError` thrown inside a method is caught by the framework and surfaced as a failed dispatch with the error's `message` as the `reason`. Other throws bubble.

The raw `DeliveryAdapter` contract remains legal for adapters that need to escape every helper convention (in-house drivers pushing onto a Redis queue, providers with non-fetch transports). `createDeliveryAdapter` produces a `DeliveryAdapter` and is fully interoperable.

Worked examples:

- [`docs/examples/parcel-adapter.md`](../examples/parcel-adapter.md) — OAuth2 token caching, branch-bound credentials, region-routed cancel endpoints, `quoteDelivery` + `getAirwayBill`.
- [`docs/examples/leajlak-adapter.md`](../examples/leajlak-adapter.md) — long-lived bearer token, non-standard `CANCEL` HTTP verb, provider-side `shop_id` mapped from the branch.
- [`docs/examples/qmile-adapter.md`](../examples/qmile-adapter.md) — driver selection via `listDrivers` + `preferredDriverId`, fire-and-forget dispatch (no webhook), provider-returned price & ETA at create time.

### Extension points (`extends`)

The capability flags cover behaviors common to multiple providers. Anything genuinely provider-specific — QMile's `packageSize` enum, a carrier's `customsDeclaration` field, a hyperlocal provider's `tipAmount` — lives in the adapter's `extends` map. Each well-known key adds a typed extension to a specific framework seam; the schema's inferred output is then visible both inside the adapter and at the SDK call site.

```ts
createDeliveryAdapter({
  id: "qmile",
  options: qmileOptions,
  capabilities: { /* … */ },

  extends: {
    "checkout:input": v.object({
      parcel: v.optional(v.object({
        size: v.optional(v.picklist(["small", "medium", "large", "extra_large"])),
        weight: v.optional(v.object({ value: v.number(), unit: v.picklist(["g", "kg"]) })),
        fragile: v.optional(v.boolean()),
      })),
    }),
  },

  createDelivery: async ({ fetch, ctx }) => {
    // ctx.checkoutInput is typed as v.InferOutput<extends["checkout:input"]>
    const size = ctx.checkoutInput?.parcel?.size ?? "medium"
    // …
  },
})
```

**Active extension points (v1):**

| Key | Where it surfaces | Read in adapter as |
|---|---|---|
| `"checkout:input"` | `commerce.orders.checkout({ delivery: { [adapterId]: ... } })` | `ctx.checkoutInput` |

**Reserved keys (planned, not yet active — listed so the namespace doesn't collide):**

| Key | Where it would surface |
|---|---|
| `"method:metadata"` | Typed shape of `deliveryMethod.metadata` per adapter |
| `"cancel:input"` | Extra knobs at `commerce.delivery.cancel(...)` |
| `"track:input"` | Query knobs at `commerce.delivery.track(...)` |
| `"webhook:context"` | Adapter-typed context surfaced on `delivery:*` events to plugins |

At checkout, the SDK exposes one optional key per registered adapter id. The value behind each key is exactly that adapter's `extends["checkout:input"]` schema's inferred output:

```ts
// With qmile() registered in deliveries: []
await commerce.orders.checkout({
  items: cart.items,
  delivery: {
    methodId: "qmile-std",
    qmile: { parcel: { size: "large", fragile: true } },
    // leajlak: never — leajlak declares no extends["checkout:input"]
  },
  payment: { adapterId: "moyasar", providerInput: { token } },
})
```

Type inference sketch:

```ts
type DeliveryCheckoutInput<TAdapters extends readonly DeliveryAdapter[]> =
  & { methodId: string }
  & {
      [A in TAdapters[number] as A['id']]?:
        A extends { extends: { "checkout:input": infer S extends StandardSchemaV1 } }
          ? StandardSchemaV1.InferOutput<S>
          : never
    }
```

**Resolution and validation:**

- Framework resolves `methodId` → method's adapter id at checkout time.
- The matching adapter's `extends["checkout:input"]` schema validates only the corresponding key (`qmile` for a qmile method, etc.).
- Other adapter keys present at the call site are dropped silently in lax mode (default), or rejected with `CommerceValidationError` when `delivery.strictCheckoutInput: true` is set in `createCommerce`.
- Adapters that don't declare `extends["checkout:input"]` get `ctx.checkoutInput: undefined`.

**Capability-gated inputs vs extension inputs.** First-class capability-gated inputs (`preferredDriverId`, `vehicleHint`, `scheduledPickupAt`, `cashOnDelivery`, `stops`) stay on `CreateDeliveryContext` — they are common enough to deserve typed fields gated by capabilities. `extends["checkout:input"]` is for everything else: adapter-specific axes that aren't worth a generic capability flag (yet).

### Idempotency

`Idempotency-Key` is an [IETF draft standard](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) and the de-facto convention across Stripe, Square, PayPal, Shopify, AWS, GitHub, Razorpay, and others. The framework forwards `ctx.request.idempotencyKey` on every outgoing POST/PUT/PATCH via `createDeliveryAdapter`'s default `fetch` shell — adapters do not write the header by hand.

```ts
// Built-in behavior of createDeliveryAdapter.fetch:
// every request gets these headers unless the adapter overrides:
//   Idempotency-Key: <ctx.request.idempotencyKey>
```

Per-adapter overrides for providers that rename the header:

```ts
createDeliveryAdapter({
  id: "exotic-provider",
  idempotency: {
    header: "X-Provider-Request-Id",   // override the default 'Idempotency-Key'
    methods: ["POST", "PUT"],          // optional; default is ["POST", "PUT", "PATCH"]
    // set to `false` to disable forwarding entirely for providers that 5xx on unknown headers:
    // enabled: false,
  },
  // …
})
```

Receivers that don't recognize the header treat it as opaque metadata; receivers that do treat repeat deliveries of the same key as one operation. The cost of being wrong is zero on indifferent providers and catastrophic-failure-prevention on aware ones.

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
  adapter: string                       // references the delivery adapter registry (literal-typed)
  providerAccountRef: string | null     // provider-side identifier for this method (shopId, branch code, merchant number, etc.)
  pricing: jsonb                        // tagged strategy config: { strategy: '<id>', ...settings }
  name: string
  enabled: boolean
  minOrderAmount: Money | null
  maxDistanceMeters: number | null
  autoDispatch: 'inherit' | 'auto' | 'manual'   // override the merchant/app default; 'inherit' is the default

  merchant: merchant().optional()       // hierarchical tenancy
  branch: branch().optional()           // per-branch pricing/methods

  metadata: jsonb                       // adapter-specific extras (packageSize for QMile, pickup contact for Parcel, etc.)
  createdAt: Date
  updatedAt: Date
}
```

The `pricing` column stores a tagged shape — `{ strategy: 'distance-based', ...typedSettings }` — produced by strategy factories. The factory is the input format for create/update; the persisted JSON carries the strategy tag so calculation can look up the right strategy later.

**`providerAccountRef` vs `metadata`.** `providerAccountRef` is the canonical place for "the identifier the provider uses to recognize this method" — Leajlak's `shopId`, a carrier's merchant account number, a region code. Adapters read it as `ctx.method.providerAccountRef`. `metadata` remains the catch-all for adapter-specific extras that aren't an identifier (QMile's `packageSize` policy, Parcel's pickup contact name). Having one canonical field prevents every adapter from coining its own metadata key for the same concept.

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
pending ─► dispatched ─► accepted ─► picked_up ─► in_transit ─► arrived ─► delivered
              │             │           │                          │           │
              │             │           └───────────┐              │           └─► returned*
              │             │                       ▼              ▼
              ▼             ▼                   cancelled      cancelled
          cancelled     cancelled
          failed        failed                      ▼
                                                 failed

(* only when capabilities.returnFlow is true)
```

`accepted`, `picked_up`, `arrived`, and `returned` are optional — providers that don't expose them simply skip ahead (e.g., `dispatched → in_transit → delivered` is a valid trace). Forward-only jumps are accepted by the state machine; backward transitions are not.

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

The canonical `DeliveryEvent` discriminated union is defined alongside the webhook contract — see [Webhook contract](#webhook-contract) above. Core dispatches each event to the delivery state machine and to any plugin hooks subscribing to `delivery:*` events.

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
| `leajlak({ apiKey, baseUrl, webhookSecret })` | [Leajlak](https://docs.leajlak.com/) | real-time tracking, cancellation, webhooks (see [leajlak-adapter.md](../examples/leajlak-adapter.md)) |
| `parcel({ clientID, clientSecret, region, webhookSecret })` | [Parcel](https://api-docs.tryparcel.com/) | cancellation, scheduling, webhooks (see [parcel-adapter.md](../examples/parcel-adapter.md)) |
| `qmile({ apiKey, baseUrl })` | QMile | driver selection, estimatedTime (see [qmile-adapter.md](../examples/qmile-adapter.md)) |

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
