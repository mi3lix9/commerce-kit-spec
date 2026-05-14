# Moyasar payment adapter (example)

> Illustrative reference for authoring a Commerce Kit payment adapter against a real provider. Wraps the [Moyasar REST API](https://docs.moyasar.com/api/api-introduction). Uses [`createPaymentAdapter`](../architecture/50-adapter-system.md#authoring-with-createpaymentadapter); reconcile against the helper's normative rules before shipping.

## What this example shows

- The minimum a redirect-flow payment adapter needs when built on `createPaymentAdapter`: options schema, capabilities, a configured `fetch`, a `status(payload)` function, and four short method bodies.
- How an HMAC-signed webhook is verified using standard Node `crypto` primitives.
- How idempotency, integer minor units (halalas), and webhook event IDs are kept distinct from transaction references.

Tenancy: this is a **single-adapter example**. The same adapter works unchanged in single-store, multi-merchant, and marketplace (`tenancy.checkout: "split"`) configurations — the adapter is tenancy-agnostic; Commerce Kit owns tenancy routing.

## Provider summary

Moyasar is a Saudi Arabia–based PSP.

| Concern | Moyasar behavior |
|---|---|
| Base URL | `https://api.moyasar.com/v1` |
| Auth | HTTP Basic; secret key as username, empty password |
| Money | Integer minor units (halalas for SAR). Matches Commerce Kit's contract — `moneyConversion` is the identity `money.minorUnits()`. |
| Currencies | `SAR`, `USD`, `AED`, `EGP`, `KWD`, `BHD`, `QAR`, `OMR`, `JOD`. |
| Sources | `creditcard`, `applepay`, `stcpay`, `samsungpay`. |
| Auth/capture | `auto_capture: false` on create + `POST /payments/:id/capture` later. |
| Void | `POST /payments/:id/void` while `authorized`. |
| Refund | `POST /payments/:id/refund`; supports partial. |
| Webhooks | HMAC-SHA256 signature in `X-Moyasar-Signature`; event ids in payload `id`. |
| Statuses | `initiated`, `paid`, `failed`, `authorized`, `captured`, `refunded`, `voided`. |

## Adapter

```ts
import { createPaymentAdapter } from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { createHmac, timingSafeEqual } from "node:crypto"
import * as v from "valibot"

const moyasarOptions = v.object({
  secretKey: v.pipe(v.string(), v.startsWith("sk_")),
  webhookSecret: v.string(),
})

export const moyasar = createPaymentAdapter({
  id: "moyasar",
  options: moyasarOptions,

  capabilities: {
    flow: "redirect",
    supportsCapture: true,
    supportsVoid: true,
    supportsRefund: true,
    supportsPartialRefund: true,
    supportsOffSession: false,
    voidableStates: ["AUTHORIZED"],
  },

  moneyConversion: (money) => money.minorUnits(),

  fetch: ({ options }) => (input, init) =>
    globalThis.fetch(`https://api.moyasar.com/v1${input}`, {
      ...init,
      headers: {
        ...init?.headers,
        Authorization: `Basic ${btoa(options.secretKey + ":")}`,
        "Content-Type": "application/json",
      },
    }),

  status: (p) => {
    switch (p.status) {
      case "paid":       return "captured"
      case "authorized": return "authorized"
      case "refunded":   return "refunded"
      case "voided":     return "voided"
      case "initiated":
        // Moyasar uses "initiated" both for brand-new payments and for ones
        // awaiting 3-D Secure. The transaction_url presence disambiguates.
        return p.source?.transaction_url ? "requires_action" : "failed"
      case "failed":
      case "expired":    return "failed"
      default:
        throw new CommerceProviderError(
          `Unknown Moyasar status: ${p.status}`,
          { providerId: "moyasar", providerStatus: p.status },
        )
    }
  },

  authorize: async ({ fetch, ctx }) => {
    const res = await fetch("/payments", {
      method: "POST",
      body: JSON.stringify({
        amount: ctx.amount,
        currency: ctx.currency,
        description: ctx.description ?? `Order ${ctx.orderId}`,
        callback_url: ctx.returnUrl,
        source: { type: "creditcard", token: requireToken(ctx) },
        metadata: {
          order_id: ctx.orderId,
          merchant_id: ctx.merchantId ?? null,
          branch_id: ctx.branchId ?? null,
        },
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`authorize failed: ${res.status}`, {
        providerId: "moyasar", body: await res.text(),
      })
    }
    const payment = await res.json()
    return {
      payload: payment,
      paymentUrl: payment.source?.transaction_url,
      reason: payment.source?.message,
    }
  },

  capture: async ({ fetch, providerReference, amount }) => {
    const res = await fetch(`/payments/${providerReference}/capture`, {
      method: "POST",
      body: JSON.stringify({ amount }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`capture failed: ${res.status}`, {
        providerId: "moyasar", body: await res.text(),
      })
    }
    return { payload: await res.json() }
  },

  refund: async ({ fetch, providerReference, amount }) => {
    const res = await fetch(`/payments/${providerReference}/refund`, {
      method: "POST",
      body: JSON.stringify({ amount }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`refund failed: ${res.status}`, {
        providerId: "moyasar", body: await res.text(),
      })
    }
    return { payload: await res.json() }
  },

  cancel: async ({ fetch, providerReference }) => {
    const res = await fetch(`/payments/${providerReference}/void`, { method: "POST" })
    if (!res.ok) {
      throw new CommerceProviderError(`void failed: ${res.status}`, {
        providerId: "moyasar", body: await res.text(),
      })
    }
    return { payload: await res.json() }
  },

  webhook: {
    verify: ({ rawBody, headers, options }) => {
      const expected = createHmac("sha256", options.webhookSecret).update(rawBody).digest("hex")
      const provided = headers["x-moyasar-signature"] ?? ""
      const a = Buffer.from(provided, "hex")
      const b = Buffer.from(expected, "hex")
      return a.length === b.length && timingSafeEqual(a, b)
    },
    extract: (raw) => JSON.parse(raw).data,
    // deliveryKey defaults to payload.id; Moyasar's webhook event id is unique per delivery.
  },
})
```

Notes on what the helper owns for free:

- `ctx.amount` arrives as integer minor units (halalas for SAR) because `moneyConversion` is `money.minorUnits()` — no conversion math in any method.
- `ctx.idempotencyKey` is forwarded onto outgoing POSTs by the configured fetch closure (framework concern, configured once).
- `metadata.order_id` round-trips so the webhook resolves back to the Commerce Kit order without a persisted `paymentReference` field.
- Each method throws `CommerceProviderError` on non-2xx; the framework catches it and produces `outcome: 'failed'` with the error's message as the `reason`.
- The `status(payload)` function is the single source of truth for outcome derivation — every method's payload flows through it. `requires_action` is decided by inspecting `source.transaction_url`, not the status string alone.

## Usage

```ts
// lib/commerce.ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { moyasar } from "@commerce-kit/moyasar"
import { db, schema } from "./db"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [
    moyasar({
      secretKey: process.env.MOYASAR_SECRET_KEY!,
      webhookSecret: process.env.MOYASAR_WEBHOOK_SECRET!,
    }),
  ],
})
```

Checkout selects the adapter by its literal id:

```ts
await commerce.orders.checkout({
  items: cart.items,
  payment: { adapterId: "moyasar", providerInput: { token } },
})
```

Webhook mount: `POST /webhooks/payment/moyasar`. Framework adapter rules: [80-framework-adapters.md](../architecture/80-framework-adapters.md).

Multiple Moyasar accounts (e.g. one per merchant) — `id` overrides at the call site, not in the options schema:

```ts
payments: [
  moyasar({ id: "moyasar-sa", secretKey: env.MOYASAR_SA_KEY, webhookSecret: env.MOYASAR_SA_HOOK }),
  moyasar({ id: "moyasar-ae", secretKey: env.MOYASAR_AE_KEY, webhookSecret: env.MOYASAR_AE_HOOK }),
]
```

The compile-time duplicate-id check catches collisions at the `createCommerce()` call site.

## Test plan

Authorize:
- token-based credit card returns `outcome: 'requires_action'` with `paymentUrl` set when 3-D Secure is required; no transaction row inserted.
- token-based credit card returns `outcome: 'captured'` when Moyasar auto-captures; one `CAPTURE` row appended.
- declined card returns `outcome: 'failed'` with `reason` carrying Moyasar's message; no transaction row; order transitions to `failed`.
- unsupported currency rejects synchronously before any network call (Standard Schema validates options at startup; the schema can be extended with a runtime currency guard).
- `ctx.idempotencyKey` is forwarded; identical replay returns the original payment.

Capture / void / refund:
- partial capture amounts are forwarded as integer halalas (no conversion math); one `CAPTURE` row appended.
- a `VOID` against a transaction whose latest type is `AUTHORIZE` returns `outcome: 'voided'` and appends a `VOID` row.
- partial refund returns `outcome: 'refunded'` and appends one `REFUND` row; existing rows untouched. Core derives `PARTIALLY_REFUNDED` from `Σrefund < Σcapture`.
- subsequent refund that closes the gap pushes the derived state to `REFUNDED` without any adapter awareness.

Webhook:
- valid signature parses; framework runs `status(payload)` and inserts the right row.
- tampered body or wrong secret rejects with `CommerceWebhookError`.
- replay of the same `event.id` is a no-op at core's webhook idempotency layer (default `deliveryKey: (p) => p.id`).
- unknown status throws from `status()` and surfaces as a `provider_error` log entry rather than silently failing.

Tenancy:
- `merchant_id` and `branch_id` appear in Moyasar `metadata` when set; single-store config sends explicit `null`s — no schema drift between tenancy modes.

Money:
- `1234` halalas stays `1234` end-to-end. No `* 100`, no `/ 100`, no `Number` parsing — `money.minorUnits()` is the identity.
- mixed-currency cart is rejected before reaching `authorize` (core invariant; adapter must not paper over it).
