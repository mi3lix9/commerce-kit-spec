# QMile delivery adapter (example)

> Illustrative reference for authoring a Commerce Kit delivery adapter against a real provider. Wraps the QMile External Orders API. Uses [`createDeliveryAdapter`](../architecture/54-delivery-adapters.md#authoring-with-createdeliveryadapter); reconcile against the helper's normative rules before shipping.
>
> Reconciled against the QMile partner-supplied integration notes (`POST /api/drivers/available`, `POST /api/orders/external`) shared 2026-05-14. QMile's public docs do not currently expose cancellation, tracking, or webhook endpoints — those capabilities are declared `false`. If QMile adds them, extend the adapter rather than working around their absence.

## What this example shows

- **`driverSelection` end-to-end.** QMile is the first integration where the merchant can hand-pick a driver. The adapter implements `listDrivers` (so the SDK exposes `commerce.delivery.listDrivers(...)`) and reads `ctx.preferredDriverId` in `createDelivery`. Omitting it falls back to QMile's auto-assign.
- **Provider-returned ETA + fee at create time.** QMile's create response carries `price`, `distance`, and `estimatedTime` (minutes). The adapter normalizes both into `Money` and a `Date` so plugins can reconcile QMile's quoted price against the strategy-computed delivery fee.
- **Fire-and-forget dispatch.** QMile doesn't expose post-create status today — no cancel, no track, no webhook. The capability flags reflect this. The `Delivery` row stays in `dispatched` until an admin transitions it manually via `commerce.delivery.transition(...)` (or until a custom plugin polls QMile's dashboard / status feed if/when one appears).

## Provider summary

| Concern | QMile behavior |
|---|---|
| Base URL | Pass per-environment via `options.baseUrl` (the partner-shared examples reference a Replit-hosted host; QMile partners are issued a stable URL). |
| Auth | `Authorization: Bearer <apiKey>`; single long-lived API key. |
| Money | `price` returned as a `Number` in SAR major units. Currency implicit; never sent on the wire. |
| List drivers | `POST /api/drivers/available` — `{ drivers: [{ id, name, activeOrders }] }` |
| Create order | `POST /api/orders/external` |
| Cancel / track / webhook | Not exposed in the partner-shared docs. |
| Order ID | `id` is the internal QMile id; `orderNumber` is the customer-facing tracking number. The adapter uses `id` as `providerReference` and surfaces `orderNumber` as `customerTrackingNumber`. |
| Package size | `packageSize: "small" | "medium" | "large" | "extra_large"` — a QMile-specific field, exposed at checkout via `extends["checkout:input"]`. |

### Create order request body (canonical)

```json
{
  "recipientName": "John Doe",
  "recipientPhone": "+966501234567",
  "pickupLocation": "123 Main St, Dammam",
  "pickupCoordinates":  { "lat": 26.394490, "lng": 50.114187 },
  "deliveryLocation": "456 Market St, Dammam",
  "deliveryCoordinates": { "lat": 26.384860, "lng": 50.159196 },
  "packageSize": "medium",
  "notes": "Please handle with care",
  "driverId": 42
}
```

### Create order response

```json
{
  "id": 100123,
  "orderNumber": "QM-2026-100123",
  "status": "pending",
  "price": 22.50,
  "distance": 4.2,
  "estimatedTime": 28
}
```

## Declared capabilities

```ts
capabilities: {
  realtimeTracking: false,      // no read endpoint documented
  cancellation: false,          // no cancel endpoint documented
  proofOfDelivery: false,
  scheduling: false,
  estimatedTime: true,          // response.estimatedTime (minutes)
  cashOnDelivery: false,
  multiStop: false,
  vehicleSelection: false,      // packageSize is per-method config, not a per-order input
  airwayBill: false,
  driverContact: false,         // GET driver endpoint not exposed; listDrivers returns name only
  driverSelection: true,        // ⭐ the headline capability
  quotation: false,             // price arrives at create time, not as a separate quote
  returnFlow: false,
}
```

## Adapter

```ts
import { createDeliveryAdapter } from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { Money } from "commerce-kit/money"
import * as v from "valibot"

const qmileOptions = v.object({
  apiKey: v.string(),
  baseUrl: v.pipe(v.string(), v.url()),
})

type QmilePackageSize = "small" | "medium" | "large" | "extra_large"

type QmileDriver = {
  id: number
  name: string
  activeOrders: number
}

type QmileOrder = {
  id: number
  orderNumber: string
  status: "pending" | string
  price: number
  distance: number
  estimatedTime: number          // minutes
}

export const qmile = createDeliveryAdapter({
  id: "qmile",
  options: qmileOptions,

  capabilities: {
    realtimeTracking: false,
    cancellation: false,
    proofOfDelivery: false,
    scheduling: false,
    estimatedTime: true,
    cashOnDelivery: false,
    multiStop: false,
    vehicleSelection: false,
    airwayBill: false,
    driverContact: false,
    driverSelection: true,
    quotation: false,
    returnFlow: false,
  },

  // Provider-specific axes that aren't worth a generic capability flag.
  // The schema's inferred output is visible inside createDelivery as
  // ctx.checkoutInput, and at the SDK call site under `delivery.qmile`.
  extends: {
    "checkout:input": v.object({
      parcel: v.optional(v.object({
        size: v.optional(v.picklist(["small", "medium", "large", "extra_large"])),
        weight: v.optional(v.object({
          value: v.number(),
          unit: v.picklist(["g", "kg"]),
        })),
        fragile: v.optional(v.boolean()),
      })),
    }),
  },

  fetch: ({ options }) => (input, init) =>
    globalThis.fetch(`${options.baseUrl}${input}`, {
      ...init,
      headers: {
        ...init?.headers,
        Authorization: `Bearer ${options.apiKey}`,
        "Content-Type": "application/json",
      },
    }),

  listDrivers: async ({ fetch }) => {
    const res = await fetch("/api/drivers/available", { method: "POST" })
    if (!res.ok) {
      throw new CommerceProviderError(`listDrivers failed: ${res.status}`, {
        providerId: "qmile", body: await res.text(),
      })
    }
    const body = await res.json() as { drivers: QmileDriver[] }
    return body.drivers.map((d) => ({
      id: String(d.id),                    // DriverInfo.id is a string; QMile uses numeric ids
      name: d.name,
      currentLoad: d.activeOrders,
    }))
  },

  createDelivery: async ({ fetch, ctx }) => {
    // packageSize is QMile-specific, exposed at checkout via
    // extends["checkout:input"]. ctx.checkoutInput is typed against the
    // schema above — no casts needed.
    const packageSize = ctx.checkoutInput?.parcel?.size ?? "medium"

    const res = await fetch("/api/orders/external", {
      method: "POST",
      body: JSON.stringify({
        recipientName: ctx.recipient.name,
        recipientPhone: ctx.recipient.phone,
        pickupLocation: ctx.branchAddress?.formatted ?? "",
        pickupCoordinates: ctx.branchAddress?.coordinates,
        deliveryLocation: ctx.deliveryAddress.formatted,
        deliveryCoordinates: ctx.deliveryAddress.coordinates,
        packageSize,
        notes: ctx.order.notes ?? "",
        // preferredDriverId is a string in the normalized contract; QMile
        // expects a number. Cast at the wire boundary. Omitted → auto-assign.
        driverId: ctx.preferredDriverId ? Number(ctx.preferredDriverId) : undefined,
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`createDelivery failed: ${res.status}`, {
        providerId: "qmile", body: await res.text(),
      })
    }
    const body = await res.json() as QmileOrder

    return {
      providerReference: String(body.id),
      customerTrackingNumber: body.orderNumber,                      // ⬅ first-class now, not metadata
      state: "dispatched",
      fee: Money.from(body.price, ctx.order.currency),               // SAR major units → Money
      estimatedArrival: new Date(Date.now() + body.estimatedTime * 60_000),
      metadata: { distanceKm: body.distance },
      raw: body,
    }
  },

  // No cancelDelivery, trackDelivery, or webhook — capabilities forbid them.
})
```

Notes on what the helper owns for free:

- `ctx.recipient.name`, `ctx.recipient.phone`, `ctx.deliveryAddress` arrive pre-computed.
- `ctx.preferredDriverId` is gated by `capabilities.driverSelection: true` — the framework only populates it for adapters that opt in, so the conditional cast is the only piece of QMile-specific glue the adapter writes.
- Throwing `CommerceProviderError` on non-2xx surfaces as `state: 'failed'` with the message as `reason`. No special handling needed.

What the example **does not** cover and why:

- **Post-create lifecycle.** QMile's docs don't expose cancel/track/webhook. The `Delivery` row sits in `dispatched` until something else transitions it. Acceptable for fire-and-forget integrations; if QMile adds endpoints, flip the capability flags and add the methods — no contract changes needed.

## Usage

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { qmile } from "@commerce-kit/qmile"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [/* … */],

  deliveries: [
    qmile({
      apiKey: process.env.QMILE_API_KEY!,
      baseUrl: process.env.QMILE_BASE_URL!,
    }),
  ],
})
```

One method, package size chosen per order at checkout via the typed `qmile` extension:

```ts
await commerce.delivery.methods.create({
  adapter: "qmile",
  branch: "br_dammam",
  name: "QMile — Standard",
  pricing: distanceBased({
    basePrice: money(1200, "SAR"),
    baseDistanceMeters: 3000,
    perKm: money(60, "SAR"),
  }),
})
```

### Picking a driver + parcel details at checkout

Admin UIs and POS flows compose both first-class capability inputs (`preferredDriverId`) and the QMile-specific `extends["checkout:input"]` axis in one call:

```ts
const { drivers } = await commerce.delivery.listDrivers({ adapterId: "qmile" })
// drivers: [{ id, name, currentLoad }, …]

await commerce.orders.checkout({
  items: cart.items,
  delivery: {
    methodId: "qmile-standard",
    preferredDriverId: drivers[0].id,                       // capability-gated; omit for auto-assign
    qmile: {                                                // ⬅ extends["checkout:input"], type-checked per adapter
      parcel: { size: "large", fragile: true },
    },
  },
  payment: { adapterId: "moyasar", providerInput: { token } },
})
```

The SDK only exposes `commerce.delivery.listDrivers(...)` and the `preferredDriverId` checkout field when at least one registered delivery adapter declares `capabilities.driverSelection: true`. The `qmile` key appears under `delivery` only when an adapter with `id: "qmile"` and a non-empty `extends["checkout:input"]` is registered. Other adapter keys (e.g. `leajlak`, `parcel`) are typed against their own extensions — passing `qmile: {...}` on a Leajlak method is type-valid (since both adapters are registered) but is dropped silently in lax mode or rejected when `delivery.strictCheckoutInput` is on.

## Test plan

Listing drivers:
- `listDrivers` POSTs to `/api/drivers/available`; result maps `{ id, name, activeOrders }` → `{ id, name, currentLoad }`.
- numeric QMile id is stringified for `DriverInfo.id` to match the normalized contract.
- non-2xx throws `CommerceProviderError`; the caller sees the provider message.

Dispatch:
- `createDelivery` POSTs to `/api/orders/external` with `driverId` set when `ctx.preferredDriverId` is supplied; absent in the body when omitted.
- `packageSize` is read from `ctx.checkoutInput?.parcel?.size`; defaults to `"medium"` when the caller omits it.
- response `price` → `fee: Money` in the order's currency (SAR).
- response `estimatedTime` (minutes) → `estimatedArrival = now + minutes`.
- response `orderNumber` → top-level `customerTrackingNumber`; `distance` survives on `metadata`.

Capability gating (compile-time, not runtime):
- `commerce.delivery.cancel({ id })` is a TypeScript error when only QMile is registered.
- `commerce.delivery.track({ id })` is a TypeScript error when only QMile is registered.
- `commerce.delivery.listDrivers({ adapterId: "qmile" })` is allowed; the same call against an adapter without `driverSelection` is a TypeScript error.

Tenancy:
- A merchant with two branches sharing one QMile API key registers one adapter instance and one `deliveryMethod` per branch; `metadata.packageSize` and per-branch pricing diverge without per-call routing logic.
