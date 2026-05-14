# Leajlak delivery adapter (example)

> Illustrative reference for authoring a Commerce Kit delivery adapter against a real provider. Wraps the [Leajlak Partner API](https://docs.leajlak.com/docs/get-started/what-is-leajlak). Uses [`createDeliveryAdapter`](../architecture/54-delivery-adapters.md#authoring-with-createdeliveryadapter); reconcile against the helper's normative rules before shipping.
>
> Reconciled against the public Leajlak docs (crawled 2026-05-14). Source pages: [partner account](https://docs.leajlak.com/docs/get-started/what-is-leajlak), [bearer token](https://docs.leajlak.com/docs/get-started/GeneratingBearertoken), [create order](https://docs.leajlak.com/docs/get-started/CreateOrderApi), [get order](https://docs.leajlak.com/docs/get-started/GetOrder), [cancel order](https://docs.leajlak.com/docs/get-started/DeleteOrder), [webhook](https://docs.leajlak.com/docs/get-started/WebHook). Decimal vs integer money, the webhook signature header name, and the production base URL are ambiguous in the public docs — validate against a live sandbox before shipping.

## What's different from the Parcel example

1. **Static long-lived bearer token, no OAuth flow.** No token cache, no refresh path; the `fetch` closure just sets the `Authorization` header.
2. **Non-standard `CANCEL` HTTP verb.** Documented as `CANCEL /orders/{id}` — not `DELETE` or `PUT`. Passed through `globalThis.fetch` verbatim; some legacy proxies and serverless runtimes normalize the method and will need a workaround.
3. **Provider-side `shop_id` mapped from the branch** via `deliveryMethod.metadata.shopId`. Populated once at onboarding from `GET /api/partner/shop`.
4. **No quote endpoint, no AWB, no driver selection** — the capability flags reflect this.

## Provider summary

| Concern | Leajlak behavior |
|---|---|
| Base URL | Pass per-environment via `options.baseUrl` (staging admin: `https://staging.4ulogistic.com`). |
| Auth | `Authorization: Bearer <token>`; one long-lived token per partner. |
| Money | `order.total` is a `Number`; decimal vs integer is not specified. Working assumption: SAR **major units** as a decimal (e.g. `12.50`). Validate against a sandbox. |
| Currency | Implicit (SAR). Never sent on the wire. |
| Create order | `POST /orders` |
| Get order | `GET /orders/{id}` — `{ id, status, dsp_order_id, driver? }` |
| Cancel order | `CANCEL /orders/{id}` — returns 202; rejected post-pickup. |
| List shops | `GET /api/partner/shop` — onboarding-time call, used to populate `metadata.shopId`. |
| Webhook auth | Partner-configured `key`; transport header name not documented (assumed `x-leajlak-key`). |
| Webhook body | `{ id, dsp_order_id, status, driver: { name, phone } }` |
| Order ID | `id` is the client-supplied unique ID (round-trips); `dsp_order_id` is Leajlak's reference. |
| Statuses | `New Order`, `Order Accept`, `Start Ride`, `Reached Shop`, `Order Picked`, `Shipped`, `Delivered`, `Cancelled` |

## Declared capabilities

```ts
capabilities: {
  realtimeTracking: true,       // GET /orders/{id} returns driver.location once assigned
  cancellation: true,           // with a runtime carve-out after Order Picked
  proofOfDelivery: false,       // not surfaced by the public docs
  scheduling: false,            // no scheduled-pickup field
  estimatedTime: false,         // createDelivery does not return an ETA
  cashOnDelivery: true,         // order.payment_type = 1
  multiStop: false,             // single-stop API
  vehicleSelection: false,      // no vehicle field
  airwayBill: false,            // no AWB endpoint
  driverContact: true,          // driver name + phone in GET and webhook
  driverSelection: false,       // Leajlak assigns captains automatically
  quotation: false,             // no quote endpoint
  returnFlow: false,            // no return-to-sender exposed
}
```

## Status semantics

| Leajlak status | `DeliveryState` | Notes |
|---|---|---|
| `New Order` | (not emitted) | Initial state on `GET /orders/{id}` before assignment. |
| `Order Accept` | `accepted` | Captain assigned and acknowledged. |
| `Start Ride` | `in_transit` | Captain en route to the shop. |
| `Reached Shop` | `in_transit` | At pickup, not yet picked up. |
| `Order Picked` | `picked_up` | Cancellation is no longer accepted. |
| `Shipped` | `in_transit` | En route to recipient. |
| `Delivered` | `delivered` | Terminal. |
| `Cancelled` | `cancelled` | Terminal. |

A real `Failed` status is not documented; `toState` throws on unknowns so a previously-unseen value loudly surfaces rather than silently advancing the row.

## Adapter

```ts
import { createDeliveryAdapter } from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { timingSafeEqual } from "node:crypto"
import * as v from "valibot"

const leajlakOptions = v.object({
  apiKey: v.string(),
  baseUrl: v.pipe(v.string(), v.url()),
  webhookSecret: v.string(),
  webhookSignatureHeader: v.optional(v.string(), "x-leajlak-key"),    // per-tenant override
})

type LeajlakStatus =
  | "New Order" | "Order Accept" | "Start Ride" | "Reached Shop"
  | "Order Picked" | "Shipped" | "Delivered" | "Cancelled"

type LeajlakOrder = {
  id: string
  dsp_order_id?: number | string
  status: LeajlakStatus
  driver?: {
    name: string
    phone: string
    location?: { latitude: string | number; longitude: string | number }
  }
}

function toState(status: LeajlakStatus): import("commerce-kit").DeliveryState {
  switch (status) {
    case "New Order":     return "dispatched"
    case "Order Accept":  return "accepted"
    case "Start Ride":
    case "Reached Shop":  return "in_transit"
    case "Order Picked":  return "picked_up"
    case "Shipped":       return "in_transit"
    case "Delivered":     return "delivered"
    case "Cancelled":     return "cancelled"
    default:
      throw new CommerceProviderError(`Unknown Leajlak status: ${status as string}`, {
        providerId: "leajlak", providerStatus: status,
      })
  }
}

function toDriver(order: LeajlakOrder) {
  if (!order.driver) return undefined
  const loc = order.driver.location
  return {
    // Leajlak does not expose a stable driver id in the public docs; use the
    // captain's phone as a stable-enough handle so DriverInfo.id is satisfied.
    // Inert anyway — capabilities.driverSelection is false, so the id never
    // round-trips back into createDelivery.
    id: order.driver.phone,
    name: order.driver.name,
    phone: order.driver.phone,
    currentLocation: loc ? { lat: Number(loc.latitude), lng: Number(loc.longitude) } : undefined,
  }
}

export const leajlak = createDeliveryAdapter({
  id: "leajlak",
  options: leajlakOptions,

  capabilities: {
    realtimeTracking: true,
    cancellation: true,
    proofOfDelivery: false,
    scheduling: false,
    estimatedTime: false,
    cashOnDelivery: true,
    multiStop: false,
    vehicleSelection: false,
    airwayBill: false,
    driverContact: true,
    driverSelection: false,
    quotation: false,
    returnFlow: false,
  },

  fetch: ({ options }) => (input, init) =>
    globalThis.fetch(`${options.baseUrl}${input}`, {
      ...init,
      headers: {
        ...init?.headers,
        Authorization: `Bearer ${options.apiKey}`,
        Accept: "application/json",
        "Content-Type": "application/json",
      },
    }),

  createDelivery: async ({ fetch, ctx }) => {
    const shopId = ctx.method.providerAccountRef
    if (!shopId) {
      throw new CommerceProviderError("leajlak delivery method is missing providerAccountRef (the Leajlak shop id)", {
        providerId: "leajlak",
      })
    }

    const res = await fetch("/orders", {
      method: "POST",
      body: JSON.stringify({
        id: ctx.order.id,
        shop_id: shopId,
        delivery_details: {
          name: ctx.recipient.name,
          phone: ctx.recipient.phone,
          coordinate: ctx.deliveryAddress.coordinates && {
            latitude: ctx.deliveryAddress.coordinates.lat,
            longitude: ctx.deliveryAddress.coordinates.lng,
          },
          address: ctx.deliveryAddress.formatted,
        },
        order: {
          payment_type: ctx.cashOnDelivery ? 1 : 0,            // 0 prepaid · 1 COD · 10 swiping
          total: ctx.cashOnDelivery
            ? ctx.cashOnDelivery.toMajorUnits()
            : ctx.order.totals.grandTotal.toMajorUnits(),
          notes: ctx.order.notes ?? "",
        },
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`createDelivery failed: ${res.status}`, {
        providerId: "leajlak", body: await res.text(),
      })
    }
    const body = await res.json() as LeajlakOrder
    return {
      providerReference: body.id ?? ctx.order.id,
      customerTrackingNumber: body.dsp_order_id ? String(body.dsp_order_id) : undefined,
      state: "dispatched",
      driver: toDriver(body),                                  // usually undefined at create time
      raw: body,
    }
  },

  cancelDelivery: async ({ fetch, providerReference }) => {
    // Non-standard verb passed through verbatim.
    const res = await fetch(`/orders/${providerReference}`, { method: "CANCEL" })
    if (res.status === 202) {
      return { providerReference, state: "cancelled", raw: { ok: true } }
    }
    const body = await res.text().catch(() => "")
    // Documented post-pickup rejection: keep the row's current state by
    // returning state: 'failed' rather than 'cancelled'. The classifier picks
    // 'refused' for the "already in transit" path, 'provider_error' otherwise.
    const code = body.includes("already in transit") ? "refused" : "provider_error"
    return {
      providerReference,
      state: "failed",
      reason: body,
      failureReason: { code, message: body },
      raw: body,
    }
  },

  trackDelivery: async ({ fetch, providerReference }) => {
    const res = await fetch(`/orders/${providerReference}`)
    if (!res.ok) {
      throw new CommerceProviderError(`trackDelivery failed: ${res.status}`, {
        providerId: "leajlak", body: await res.text(),
      })
    }
    const body = await res.json() as LeajlakOrder
    const driver = toDriver(body)
    return {
      providerReference,
      state: toState(body.status),
      currentLocation: driver?.currentLocation,
      driver,
      raw: body,
    }
  },

  webhook: {
    signatureHeader: "x-leajlak-key",                              // default; merchants override per tenant if needed
    verify: ({ rawBody: _rawBody, headers, options }) => {
      const provided = headers[options.webhookSignatureHeader.toLowerCase()] ?? ""
      const expected = options.webhookSecret
      const a = Buffer.from(provided)
      const b = Buffer.from(expected)
      return a.length === b.length && timingSafeEqual(a, b)
    },
    extract: (raw) => JSON.parse(raw) as LeajlakOrder,
    providerReference: (p) => p.id,
    deliveryKey: (p) => `${p.id}:${p.status}:${p.dsp_order_id ?? ""}`,
    toEvent: (p) => {
      const state = toState(p.status)
      const driver = toDriver(p)
      const ref = p.id
      const now = new Date()
      switch (state) {
        case "accepted":
          return { kind: "accepted", providerReference: ref, driver, raw: p }
        case "picked_up":
          return { kind: "picked_up", providerReference: ref, pickedUpAt: now, raw: p }
        case "in_transit":
          return { kind: "in_transit", providerReference: ref, currentLocation: driver?.currentLocation, driver, raw: p }
        case "delivered":
          return { kind: "delivered", providerReference: ref, deliveredAt: now, raw: p }
        case "cancelled":
          return { kind: "cancelled", providerReference: ref, raw: p }
        case "dispatched":
          return { kind: "dispatched", providerReference: ref, raw: p }
        case "failed":
          return { kind: "failed", providerReference: ref, reason: "provider reported failure", raw: p }
        default:
          throw new CommerceProviderError(`unmapped event state: ${state}`, { providerId: "leajlak" })
      }
    },
  },
})
```

Notes on what the helper owns for free:

- `ctx.recipient`, `ctx.cashOnDelivery`, `ctx.deliveryAddress` arrive pre-computed.
- `toState` and `toDriver` are local functions; every method body that exposes state uses them, including `webhook.toEvent`.
- Each method returns the normalized shape with the provider payload preserved on `raw`. `commerce.delivery.get(id)` exposes the same shape regardless of adapter.
- `cancelDelivery`'s post-pickup rejection returns `state: 'failed'` so core preserves the row's current state instead of transitioning to `cancelled`. The caller sees the provider's message in `reason`.

## Usage

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { leajlak } from "@commerce-kit/leajlak"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [/* … */],

  deliveries: [
    leajlak({
      apiKey: process.env.LEAJLAK_API_KEY!,
      baseUrl: process.env.LEAJLAK_BASE_URL!,
      webhookSecret: process.env.LEAJLAK_WEBHOOK_KEY!,
    }),
  ],
})
```

Per-branch shopId stamped on the method:

```ts
await commerce.delivery.methods.create({
  adapter: "leajlak",
  branch: "br_riyadh",
  name: "Leajlak — Riyadh",
  providerAccountRef: "821015895",                  // ⬅ the Leajlak shop id, first-class
  pricing: distanceBased({
    basePrice: money(1700, "SAR"),
    baseDistanceMeters: 3000,
    perKm: money(100, "SAR"),
  }),
  maxDistanceMeters: 20000,
})
```

Webhook mount: `POST /webhooks/delivery/leajlak`. Configure the partner-side webhook URL + key in the Leajlak admin console so the transport `key` matches `options.webhookSecret`.

## Test plan

Dispatch:
- `createDelivery` posts the canonical body; `metadata.shopId` is the `shop_id`; missing shopId throws synchronously.
- `payment_type` is `1` when `ctx.cashOnDelivery` is set, `0` otherwise.
- `total` is the cash-collected amount when COD, else the order grand total (both in SAR major units — validate against a sandbox).
- the `Delivery` row transitions to `dispatched`; `providerReference` is `order.id`.

Cancel:
- HTTP `CANCEL` is used; runtimes that intercept the method are flagged at integration time.
- 202 → `state: 'cancelled'`.
- documented "already in transit" rejection → `state: 'failed'` with the message on `reason`; the row's prior state is preserved.

Track:
- `trackDelivery` returns the normalized shape with `state` derived from `toState`; `driver` and `currentLocation` are populated when assigned.
- `New Order` returns `state: 'dispatched'` — no transition needed if the row is already there.

Webhook → normalized event:
- `Order Accept` → `kind: 'accepted'` with `driver`.
- `Start Ride` / `Reached Shop` / `Shipped` → `kind: 'in_transit'` (with `currentLocation` when present).
- `Order Picked` → `kind: 'picked_up'`.
- `Delivered` → `kind: 'delivered'`.
- `Cancelled` → `kind: 'cancelled'`.
- mismatched key → `CommerceWebhookError`.
- replay of `(id, status, dsp_order_id)` → no-op.
- unknown status → `toState` throws; surfaces as `provider_error`.

Tenancy:
- Two branches with two `metadata.shopId` values dispatch through one adapter instance; correct shop without per-call routing logic.

Money (validate before shipping):
- `12.50` SAR sent as `12.50` survives the round trip.
- If the provider instead expects integer minor units (`1250`), swap `toMajorUnits()` for `minorUnits()` at the two call sites.
