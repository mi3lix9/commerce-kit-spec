# Parcel delivery adapter (example)

> Illustrative reference for authoring a Commerce Kit delivery adapter against a real provider. Wraps the [Parcel Delivery API v4](https://api-docs.tryparcel.com/). Uses [`createDeliveryAdapter`](../architecture/54-delivery-adapters.md#authoring-with-createdeliveryadapter); reconcile against the helper's normative rules before shipping.
>
> Reconciled against the public Parcel docs page (crawled 2026-05-14). Endpoint field shapes beyond the public introduction (request/response models per endpoint, webhook payload schema) were not exposed in machine-readable form at scrape time — implementers should validate the wire shapes against a live sandbox before shipping.

## What this example shows

- **Inputs arrive pre-normalized.** The adapter never reaches into `ctx.order.payment` or `ctx.order.metadata` — `ctx.recipient`, `ctx.cashOnDelivery`, `ctx.vehicleHint`, and `ctx.scheduledPickupAt` are computed by the framework and handed in directly.
- **Outputs are normalized.** Every method returns a Commerce Kit shape (`CreateDeliveryResult`, `TrackDeliveryResult`, `CancelDeliveryResult`, `DeliveryEvent`). The provider payload travels untouched on `raw` so plugins can introspect it; core never reads it.
- **OAuth2 token caching** lives entirely in the `fetch` closure: one token per `(region, clientID)` pair, refreshed lazily, retried once on 401.
- **Branch-bound credentials** with `id`-overridden multi-instance setup — one factory call per branch since each Parcel credential pair is region-locked.
- **Region-routed wire format.** BH-manama identifies orders by `taskRelation`; SA-riyadh and others use `id`. The adapter folds region into the right field and path; call sites never see it.

Tenancy: single-adapter example. The same source works in single-store, multi-merchant, and marketplace (`tenancy.checkout: "split"`) — Commerce Kit owns tenancy routing.

## Provider summary

| Concern | Parcel behavior |
|---|---|
| Base URL | `https://api.tryparcel.com` |
| Auth — token endpoint | HTTP Basic against `POST /oauth2/token`. Returns a bearer token. |
| Auth — everything else | `Authorization: Bearer <access_token>` |
| Credential scope | `clientID`/`clientSecret` bound to a single **branch**; each branch bound to a **region**; region defines currency. |
| Response envelope | `{ message: string, status: number, data: object }` |
| Create order | `POST /api/v4/task` |
| Calculate fee | `POST /api/v4/task/calculate-fee` (provider quote, gated by `capabilities.quotation`) |
| Get order | `GET /api/v4/task/{id}` |
| Cancel order | `PUT` — path varies by region (`taskRelation` vs `id`) |
| AWB PDF | `GET /api/v4/task/{id}/airway-bill` (binary PDF, gated by `capabilities.airwayBill`) |
| Webhook auth | Per-tenant secret echoed on each delivery; constant-time compare. |
| Order statuses | `Unassigned`, `Assigned`, `Accepted`, `In Progress`, `Arrived` (BH-manama only), `Successful`, `Failed`, `Canceled` |
| Order ID semantics | `id` for SA-riyadh; `taskRelation` for BH-manama. The adapter picks one as `providerReference`. |

## Declared capabilities

```ts
capabilities: {
  realtimeTracking: false,      // state via webhooks; no live driver coordinates
  cancellation: true,
  proofOfDelivery: false,       // AWB PDF is not a normalized POD
  scheduling: true,             // pickup.time accepts a scheduled ISO timestamp
  estimatedTime: false,         // createDelivery does not return an ETA
  cashOnDelivery: true,         // deliveries[].cashCollected
  multiStop: false,             // deliveries[] supports N stops on the wire, but core's Delivery row is single-stop today
  vehicleSelection: true,       // top-level 'vehicle' field
  airwayBill: true,             // GET /…/airway-bill returns a PDF
  driverContact: true,          // GET /…/task/{id} returns driver name & phone
  driverSelection: false,       // Parcel assigns drivers; merchants cannot pick one
  quotation: true,              // calculate-fee endpoint returns a binding quote
  returnFlow: false,            // no return-to-sender lifecycle exposed
}
```

## Status semantics

| Parcel status | `DeliveryState` (returned from methods + events) | Notes |
|---|---|---|
| `Unassigned` | (not emitted as a transition) | Initial state after `createDelivery` returned `dispatched`; no transition needed. |
| `Assigned` | `accepted` | Driver attached. |
| `Accepted` | `accepted` | Driver acknowledged. |
| `In Progress` | `in_transit` | |
| `Arrived` | `arrived` | BH-manama only. |
| `Successful` | `delivered` | |
| `Failed` | `failed` | `reason` carries the provider message. |
| `Canceled` | `cancelled` | |

## Adapter

```ts
import { createDeliveryAdapter } from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { Money } from "commerce-kit/money"
import { timingSafeEqual } from "node:crypto"
import * as v from "valibot"

const parcelOptions = v.object({
  clientID: v.string(),
  clientSecret: v.string(),
  region: v.picklist(["BH-manama", "SA-riyadh"]),
  webhookSecret: v.string(),
  webhookSignatureHeader: v.optional(v.string(), "x-parcel-signature"),  // per-tenant override
})

type ParcelOptions = v.InferOutput<typeof parcelOptions>

type ParcelTask = {
  id: string
  taskRelation: string
  status: "Unassigned" | "Assigned" | "Accepted" | "In Progress" | "Arrived" | "Successful" | "Failed" | "Canceled"
  driver?: { id: string; name: string; phone: string; location?: { lat: number; lng: number } }
  fee?: { amount: number; currency: string }
  estimatedArrival?: string
  statusUpdatedAt: string
}

const tokenCache = new Map<string, { token: string; expiresAt: number }>()

async function getAccessToken(options: ParcelOptions): Promise<string> {
  const cacheKey = `${options.region}:${options.clientID}`
  const cached = tokenCache.get(cacheKey)
  if (cached && cached.expiresAt > Date.now() + 30_000) return cached.token

  const res = await globalThis.fetch("https://api.tryparcel.com/oauth2/token", {
    method: "POST",
    headers: {
      Authorization: `Basic ${btoa(`${options.clientID}:${options.clientSecret}`)}`,
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: "grant_type=client_credentials",
  })
  if (!res.ok) {
    throw new CommerceProviderError(`parcel oauth2 failed: ${res.status}`, {
      providerId: "parcel", body: await res.text(),
    })
  }
  const body = await res.json() as { access_token: string; expires_in: number }
  tokenCache.set(cacheKey, {
    token: body.access_token,
    expiresAt: Date.now() + body.expires_in * 1000,
  })
  return body.access_token
}

// Local mapper — single source of truth for state derivation across methods + webhook.
function toState(status: ParcelTask["status"]): import("commerce-kit").DeliveryState {
  switch (status) {
    case "Unassigned":
    case "Assigned":
    case "Accepted":     return "accepted"
    case "In Progress":  return "in_transit"
    case "Arrived":      return "arrived"
    case "Successful":   return "delivered"
    case "Failed":       return "failed"
    case "Canceled":     return "cancelled"
    default:
      throw new CommerceProviderError(`Unknown Parcel status: ${(status as string)}`, {
        providerId: "parcel", providerStatus: status,
      })
  }
}

function toDriver(task: ParcelTask) {
  if (!task.driver) return undefined
  return {
    id: task.driver.id,
    name: task.driver.name,
    phone: task.driver.phone,
    currentLocation: task.driver.location,
  }
}

function providerReferenceOf(task: ParcelTask, region: ParcelOptions["region"]): string {
  return region === "BH-manama" ? task.taskRelation : task.id
}

export const parcel = createDeliveryAdapter({
  id: "parcel",
  options: parcelOptions,

  capabilities: {
    realtimeTracking: false,
    cancellation: true,
    proofOfDelivery: false,
    scheduling: true,
    estimatedTime: false,
    cashOnDelivery: true,
    multiStop: false,
    vehicleSelection: true,
    airwayBill: true,
    driverContact: true,
    driverSelection: false,
    quotation: true,
    returnFlow: false,
  },

  fetch: ({ options }) => async (input, init) => {
    const call = async (token: string) => globalThis.fetch(`https://api.tryparcel.com${input}`, {
      ...init,
      headers: {
        ...init?.headers,
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json",
      },
    })
    const res = await call(await getAccessToken(options))
    if (res.status !== 401) return res
    tokenCache.delete(`${options.region}:${options.clientID}`)
    return call(await getAccessToken(options))
  },

  createDelivery: async ({ fetch, ctx, options }) => {
    const res = await fetch("/api/v4/task", {
      method: "POST",
      body: JSON.stringify({
        type: 0,
        vehicle: ctx.vehicleHint ?? "car",
        pickup: {
          name: ctx.branchAddress?.notes ?? ctx.method.metadata?.pickupName,
          phone: ctx.method.metadata?.pickupPhone,
          time: ctx.scheduledPickupAt?.toISOString(),
          address: {
            fullAddress: ctx.branchAddress?.formatted,
            location: ctx.branchAddress?.coordinates,
          },
        },
        deliveries: [{
          name: ctx.recipient.name,
          phone: ctx.recipient.phone,
          address: {
            fullAddress: ctx.deliveryAddress.formatted,
            location: ctx.deliveryAddress.coordinates,
          },
          cashCollected: ctx.cashOnDelivery?.minorUnits(),
        }],
        externalRef: ctx.order.id,
        metadata: {
          orderId: ctx.order.id,
          merchantId: ctx.order.merchantId ?? null,
          branchId: ctx.order.branchId ?? null,
        },
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`createDelivery failed: ${res.status}`, {
        providerId: "parcel", body: await res.text(),
      })
    }
    const { data } = await res.json() as { data: ParcelTask }
    return {
      providerReference: providerReferenceOf(data, options.region),
      // Whichever field is NOT the operational ref is still useful as the
      // customer-facing tracking number (it's what shows up in Parcel's
      // recipient SMS).
      customerTrackingNumber: options.region === "BH-manama" ? data.id : data.taskRelation,
      state: "dispatched",
      estimatedArrival: data.estimatedArrival ? new Date(data.estimatedArrival) : undefined,
      driver: toDriver(data),
      fee: data.fee && Money.from(data.fee.amount, data.fee.currency),
      raw: data,
    }
  },

  cancelDelivery: async ({ fetch, providerReference, options, reasonCode, reason }) => {
    const path = options.region === "BH-manama"
      ? `/api/v4/task/cancel/${providerReference}`
      : `/api/v4/task/${providerReference}/cancel`
    const res = await fetch(path, {
      method: "PUT",
      body: JSON.stringify({
        reason_code: reasonCode ?? "merchant_request",
        reason: reason ?? "merchant cancelled",
      }),
    })
    if (!res.ok) {
      const body = await res.text()
      return {
        providerReference,
        state: "failed",
        reason: body,
        failureReason: { code: "provider_error", message: body },
        raw: body,
      }
    }
    const payload = await res.json()
    return { providerReference, state: "cancelled", raw: payload }
  },

  quoteDelivery: async ({ fetch, ctx, options }) => {
    const res = await fetch("/api/v4/task/calculate-fee", {
      method: "POST",
      body: JSON.stringify({
        vehicle: ctx.vehicleHint ?? "car",
        pickup: { location: ctx.branchAddress?.coordinates },
        deliveries: [{ location: ctx.deliveryAddress.coordinates }],
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`quoteDelivery failed: ${res.status}`, {
        providerId: "parcel", body: await res.text(),
      })
    }
    const { data } = await res.json() as { data: { fee: { amount: number; currency: string }; quoteId?: string; estimatedArrival?: string; vehicle?: string } }
    return {
      fee: Money.from(data.fee.amount, data.fee.currency),
      quoteId: data.quoteId,
      estimatedArrival: data.estimatedArrival ? new Date(data.estimatedArrival) : undefined,
      vehicleType: data.vehicle as import("commerce-kit").VehicleType | undefined,
      raw: data,
    }
  },

  getAirwayBill: async ({ fetch, providerReference }) => {
    const res = await fetch(`/api/v4/task/${providerReference}/airway-bill`, {
      headers: { Accept: "application/pdf" },
    })
    if (!res.ok) {
      throw new CommerceProviderError(`getAirwayBill failed: ${res.status}`, {
        providerId: "parcel", body: await res.text(),
      })
    }
    return {
      format: "pdf",
      data: new Uint8Array(await res.arrayBuffer()),
    }
  },

  webhook: {
    signatureHeader: "x-parcel-signature",                          // default; merchants override per tenant if needed
    verify: ({ rawBody: _rawBody, headers, options }) => {
      const provided = headers[(options.webhookSignatureHeader ?? "x-parcel-signature").toLowerCase()] ?? ""
      const expected = options.webhookSecret
      const a = Buffer.from(provided)
      const b = Buffer.from(expected)
      return a.length === b.length && timingSafeEqual(a, b)
    },
    extract: (raw) => JSON.parse(raw).data as ParcelTask,
    providerReference: (p) => p.taskRelation ?? p.id,
    deliveryKey: (p) => `${p.id ?? p.taskRelation}:${p.status}:${p.statusUpdatedAt}`,
    toEvent: (p) => {
      const ref = p.taskRelation ?? p.id
      const state = toState(p.status)
      const driver = toDriver(p)
      switch (state) {
        case "accepted":
          return { kind: "accepted", providerReference: ref, driver, raw: p }
        case "in_transit":
          return { kind: "in_transit", providerReference: ref, currentLocation: p.driver?.location, driver, raw: p }
        case "arrived":
          return { kind: "arrived", providerReference: ref, arrivedAt: new Date(p.statusUpdatedAt), raw: p }
        case "delivered":
          return { kind: "delivered", providerReference: ref, deliveredAt: new Date(p.statusUpdatedAt), raw: p }
        case "cancelled":
          return { kind: "cancelled", providerReference: ref, raw: p }
        case "failed":
          return { kind: "failed", providerReference: ref, reason: "provider reported Failed", raw: p }
        default:
          throw new CommerceProviderError(`unmapped event state: ${state}`, { providerId: "parcel" })
      }
    },
  },
})
```

Notes on what the helper owns for free:

- `ctx.recipient.name`, `ctx.recipient.phone`, `ctx.cashOnDelivery`, `ctx.scheduledPickupAt`, `ctx.vehicleHint` are all pre-computed — the method body never derives them.
- `toState` and `toDriver` are local functions; the framework does not run a generic status mapper. Every method body that exposes state goes through `toState`, and `toEvent` reuses both.
- Each method's `raw` field preserves the untouched payload; `commerce.delivery.get(id)` returns the normalized shape with `raw` available for plugin introspection.
- Each method throws `CommerceProviderError` on non-2xx; the framework converts to `state: 'failed'` with the error message as `reason`. `cancelDelivery` returns an explicit `failed` outcome for the documented "already in transit" case to preserve the row's current state instead of marking it as a dispatch failure.

What the example **does not** cover and why:

- **Multi-stop deliveries.** Parcel's `deliveries[]` accepts N stops, but core's `Delivery` row models one stop today. `capabilities.multiStop: false` reflects this. Multi-stop is a future RFC.
- **AWB persistence.** `getAirwayBill` returns the PDF bytes; the framework decides whether to stream-pass them to the caller or store them via the configured storage adapter and return a signed URL.

## Usage

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { parcel } from "@commerce-kit/parcel"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [/* … */],

  deliveries: [
    parcel({
      clientID: process.env.PARCEL_CLIENT_ID!,
      clientSecret: process.env.PARCEL_CLIENT_SECRET!,
      region: "SA-riyadh",
      webhookSecret: process.env.PARCEL_WEBHOOK_SECRET!,
    }),
  ],
})
```

Multiple branches → multiple factory instances with per-branch `id`s:

```ts
deliveries: [
  parcel({ id: "parcel-riyadh", clientID: env.RIY_CID, clientSecret: env.RIY_SEC, region: "SA-riyadh", webhookSecret: env.RIY_HOOK }),
  parcel({ id: "parcel-manama", clientID: env.MAN_CID, clientSecret: env.MAN_SEC, region: "BH-manama", webhookSecret: env.MAN_HOOK }),
]
```

Webhook mount: `POST /webhooks/delivery/parcel-riyadh`. Framework adapter rules: [80-framework-adapters.md](../architecture/80-framework-adapters.md).

## Test plan

Auth + token cache:
- first call hits `/oauth2/token`; subsequent calls within `expires_in - 30s` reuse the cached token.
- a downstream 401 evicts the cache entry and retries once; a second 401 surfaces as `CommerceProviderError`.
- distinct `region` keys do not share cache entries.

Dispatch:
- `createDelivery` returns `state: 'dispatched'` and `providerReference` is region-correct (`id` for SA-riyadh, `taskRelation` for BH-manama).
- `ctx.vehicleHint`, `ctx.scheduledPickupAt`, and `ctx.cashOnDelivery` appear in the wire body when set; absent when `undefined`.
- non-2xx throws; the `Delivery` row transitions to `failed` with the provider message as `reason`.

Cancel:
- `cancelDelivery` PUTs to the region-correct path; success returns `state: 'cancelled'`.
- documented "already in transit" rejection returns `state: 'failed'` (not 'cancelled'); the row's previous state is preserved.

Quote:
- `quoteDelivery` returns a `Money` fee in the branch currency; `quoteId` round-trips if the provider supplies one.

Airway bill:
- `getAirwayBill` returns `format: 'pdf'` with `data: Uint8Array`; the framework hands it back to the caller or persists via storage adapter per config.

Webhook → normalized event:
- `Assigned` / `Accepted` → `kind: 'accepted'` with `driver` populated when present.
- `In Progress` → `kind: 'in_transit'`; `currentLocation` is set when driver coordinates are in the payload.
- `Arrived` (BH-manama) → `kind: 'arrived'`.
- `Successful` → `kind: 'delivered'`.
- `Failed` / `Canceled` → `kind: 'failed'` / `'cancelled'`.
- tampered `x-parcel-signature` → `CommerceWebhookError`.
- replay of `(id, status, statusUpdatedAt)` → no-op via `deliveryKey`.
- unknown status → `toState` throws; surfaces as `provider_error` log entry.

Tenancy:
- two adapter instances → two webhook routes; each order resolves to the correct credentials with zero application-layer routing logic.
