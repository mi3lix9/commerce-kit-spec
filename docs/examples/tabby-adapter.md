# Tabby payment adapter (example)

> Companion to [`moyasar-adapter.md`](./moyasar-adapter.md). Wraps the [Tabby API](https://docs.tabby.ai/api-reference/overview) — a BNPL provider for the Gulf region. Uses [`createPaymentAdapter`](../architecture/50-adapter-system.md#authoring-with-createpaymentadapter). Reconciled against the real Tabby docs (crawled 2026-05-14); cross-check before shipping in case the contract has moved.
>
> Source pages used: [overview](https://docs.tabby.ai/api-reference/overview), [create-a-session](https://docs.tabby.ai/api-reference/checkout/create-a-session), [payment-statuses](https://docs.tabby.ai/pay-in-4-custom-integration/payment-statuses), [payment-processing](https://docs.tabby.ai/pay-in-4-custom-integration/payment-processing), [webhooks](https://docs.tabby.ai/pay-in-4-custom-integration/webhooks), [register-a-webhook](https://docs.tabby.ai/api-reference/webhooks/register-a-webhook), [technical-requirements](https://docs.tabby.ai/introduction/technical-requirements).

## What's different from the Moyasar example

Five concrete things this adapter exercises that Moyasar's doesn't:

1. **Region-specific base URLs.** Tabby uses `api.tabby.ai` for UAE/KW and `api.tabby.sa` for KSA. Same payloads on both. Selected by `options.region` inside the `fetch` closure.
2. **Decimal-string money on the wire.** Amounts are strings like `"100.50"` — up to 2 decimals for AED/SAR, up to 3 for KWD. Handled at the boundary by `moneyConversion: (money) => money.decimalString({ decimals: "currency" })`. Methods never convert.
3. **Pre-scoring at session creation.** Tabby runs a credit check at `POST /api/v2/checkout`. The session succeeds with `200` even when the customer is rejected — you detect it via `configuration.products.installments.is_available === false`. The status mapper can't see this signal (it's a session-level field, not the payment status), so the method **short-circuits with an explicit `outcome`** rather than going through `status(payload)`.
4. **Webhook authentication is a configurable header, not HMAC.** At webhook registration you supply `{ header: { title, value } }`. Tabby echoes that header on every delivery; verification is a constant-time string compare of the value.
5. **Webhooks have no `event` field.** The same payload shape is sent for authorize, capture, close, refund, expire. `status(payload)` derives the event from `status` + the lengths of `captures[]` / `refunds[]`. Out-of-order and duplicate deliveries are normal — handled by `deliveryKey` returning a composite signature.

Everything else (`createPaymentAdapter` shape, `paymentTransaction` ledger semantics, capability gating, tenancy metadata round-trip) is the same as Moyasar — cross-reference rather than restate.

## Provider summary

| Concern | Tabby behavior |
|---|---|
| Base URLs | `https://api.tabby.ai` (UAE/KW) and `https://api.tabby.sa` (KSA). Identical schemas, region routing is by merchant. |
| Auth | `Authorization: Bearer <secret_key>`; test vs live is selected by the key. |
| Money | Decimal strings: `"100.00"` for AED/SAR, `"100.000"` for KWD. |
| Currencies | `AED`, `SAR`, `KWD` (also documented: `BHD`, `QAR` for some plans). |
| Create session | `POST /api/v2/checkout` — runs pre-scoring. Returns `{ id, payment, configuration, status }`. |
| Capture | `POST /api/v2/payments/{id}/captures` — body `{ amount, reference_id }`. |
| Refund | `POST /api/v2/payments/{id}/refunds` — body `{ amount, reference_id, reason }`. Multiple partials allowed; only `CLOSED` payments are refundable. |
| Close | `POST /api/v2/payments/{id}/close` — empty body. Cancels an `AUTHORIZED` payment (no captures) or closes the remaining balance. |
| Retrieve | `GET /api/v2/payments/{id}` — full payment object, **uppercase** status. |
| Payment statuses (Retrieve) | `CREATED`, `AUTHORIZED`, `CLOSED`, `REJECTED`, `EXPIRED` |
| Webhook statuses (delivery) | Same set but **lowercase**: `authorized`, `closed`, `rejected`, `expired`. |
| Webhook auth | Custom header configured at registration. No HMAC. |
| Session expiration | 20 min default; payment moves to `EXPIRED` ~30 min after creation if no authorize event. |
| Auto-capture | If an `AUTHORIZED` payment is not captured within 21 days, Tabby auto-captures it. |

## Status semantics (the part that surprises people)

Tabby's status enum is tiny but **`CLOSED` is overloaded**. From [payment-statuses](https://docs.tabby.ai/pay-in-4-custom-integration/payment-statuses):

| Lifecycle event | Payment status after | Dashboard label |
|---|---|---|
| Session created, awaiting customer | `CREATED` | (not shown) |
| Customer completes HPP, downpayment cleared | `AUTHORIZED` | `NEW` |
| Pre-scoring or customer flow fails | `REJECTED` | (not shown, terminal) |
| Session/payment ages out | `EXPIRED` | (not shown, terminal) |
| Capture sent for the full amount | `CLOSED` | `CAPTURED` |
| `Close` after partial capture | `CLOSED` | `CAPTURED` |
| `Close` after authorize, no capture | `CLOSED` | `CANCELLED` |
| Refund(s) after capture | `CLOSED` | `REFUNDED` / `PARTIALLY REFUNDED` |

API status alone cannot tell you whether `CLOSED` means captured / cancelled / refunded. The status function disambiguates by inspecting `captures[]` and `refunds[]`.

A note on "void": Tabby is BNPL, so by the time a payment is `AUTHORIZED` the customer's downpayment has been collected. Calling `/close` on an `AUTHORIZED`-with-no-captures payment refunds the downpayment and tears down the installment plan; dashboard shows `CANCELLED`. The adapter declares `supportsVoid: true` because operationally this is "cancel before merchant settlement" — the only correct endpoint for that case (Tabby rejects `/refunds` on non-`CLOSED` payments). The semantics differ from a card-network void (no funds-released-from-hold; funds are refunded), but the merchant-side accounting outcome is the same.

## Adapter

```ts
import { createPaymentAdapter } from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { timingSafeEqual } from "node:crypto"
import * as v from "valibot"

const tabbyOptions = v.object({
  secretKey: v.pipe(v.string(), v.startsWith("sk_")),
  merchantCode: v.string(),
  region: v.picklist(["uae", "ksa"]),
  webhookHeader: v.object({
    name: v.string(),
    value: v.string(),
  }),
})

export const tabby = createPaymentAdapter({
  id: "tabby",
  options: tabbyOptions,

  capabilities: {
    flow: "redirect",
    supportsCapture: true,
    supportsVoid: true,
    supportsRefund: true,
    supportsPartialRefund: true,
    supportsOffSession: false,
    voidableStates: ["AUTHORIZED"],
  },

  moneyConversion: (money) => money.decimalString({ decimals: "currency" }),

  fetch: ({ options }) => {
    const baseURL = options.region === "ksa" ? "https://api.tabby.sa" : "https://api.tabby.ai"
    return (input, init) =>
      globalThis.fetch(`${baseURL}${input}`, {
        ...init,
        headers: {
          ...init?.headers,
          Authorization: `Bearer ${options.secretKey}`,
          "Content-Type": "application/json",
        },
      })
  },

  status: (p) => {
    const s = (p.status ?? "").toString().toUpperCase()
    if (s === "CLOSED") {
      if ((p.refunds?.length  ?? 0) > 0) return "refunded"
      if ((p.captures?.length ?? 0) > 0) return "captured"
      return "voided"
    }
    if (s === "AUTHORIZED")                  return (p.captures?.length ?? 0) > 0 ? "captured" : "authorized"
    if (s === "REJECTED" || s === "EXPIRED") return "failed"
    if (s === "CREATED")                     return "requires_action"
    throw new CommerceProviderError(
      `Unknown Tabby status: ${p.status}`,
      { providerId: "tabby", providerStatus: p.status },
    )
  },

  authorize: async ({ fetch, ctx, options }) => {
    const res = await fetch("/api/v2/checkout", {
      method: "POST",
      body: JSON.stringify({
        merchant_code: options.merchantCode,
        lang: ctx.locale ?? "en",
        merchant_urls: {
          success: ctx.returnUrl,
          cancel: ctx.cancelUrl ?? ctx.returnUrl,
          failure: ctx.failureUrl ?? ctx.returnUrl,
        },
        payment: {
          amount: ctx.amount,                        // decimal string, courtesy of money adapter
          currency: ctx.currency,
          description: ctx.description,
          buyer: requireBuyer(ctx),
          shipping_address: ctx.shippingAddress,
          order: {
            reference_id: ctx.orderId,
            tax_amount: ctx.taxAmount,
            shipping_amount: ctx.shippingAmount,
            discount_amount: ctx.discountAmount,
            items: ctx.items.map(toTabbyItem),
          },
          buyer_history: ctx.buyerHistory,           // optional but improves approval rates
          order_history: ctx.orderHistory,
          meta: {
            order_id: ctx.orderId,                   // round-trips on webhooks so core can resolve
            merchant_id: ctx.merchantId ?? null,     // the order without a persisted reference
            branch_id: ctx.branchId ?? null,
          },
        },
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`tabby checkout failed: ${res.status}`, {
        providerId: "tabby", body: await res.text(),
      })
    }
    const session = await res.json()

    // Pre-scoring rejection lives at the session level, not on payment.status — use the
    // explicit-outcome escape hatch since status(payload) can't see this signal.
    const installments = session.configuration?.products?.installments
    if (installments && installments.is_available === false) {
      return {
        outcome: "failed",
        providerReference: session.payment.id,
        reason: `tabby:${installments.rejection_reason ?? "rejected"}`,
      }
    }

    const webUrl = session.configuration?.available_products?.installments?.[0]?.web_url
    if (!webUrl) {
      throw new CommerceProviderError(
        "Tabby returned an approved session without a web_url",
        { providerId: "tabby", sessionId: session.id },
      )
    }
    return {
      payload: session.payment,
      paymentUrl: webUrl,
    }
  },

  capture: async ({ fetch, providerReference, amount }) => {
    const res = await fetch(`/api/v2/payments/${providerReference}/captures`, {
      method: "POST",
      body: JSON.stringify({
        amount,                                                          // decimal string
        reference_id: `cap-${providerReference}-${amount}`,              // idempotency key
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`tabby capture failed: ${res.status}`, {
        providerId: "tabby", body: await res.text(),
      })
    }
    return { payload: await res.json() }
  },

  refund: async ({ fetch, providerReference, amount, reason }) => {
    const res = await fetch(`/api/v2/payments/${providerReference}/refunds`, {
      method: "POST",
      body: JSON.stringify({
        amount,
        reference_id: `ref-${providerReference}-${Date.now()}`,
        reason: reason ?? "merchant_refund",
      }),
    })
    if (!res.ok) {
      throw new CommerceProviderError(`tabby refund failed: ${res.status}`, {
        providerId: "tabby", body: await res.text(),
      })
    }
    return { payload: await res.json() }
  },

  cancel: async ({ fetch, providerReference }) => {
    const res = await fetch(`/api/v2/payments/${providerReference}/close`, { method: "POST" })
    if (!res.ok) {
      throw new CommerceProviderError(`tabby close failed: ${res.status}`, {
        providerId: "tabby", body: await res.text(),
      })
    }
    return { payload: await res.json() }
  },

  webhook: {
    verify: ({ headers, options }) => {
      const provided = headers[options.webhookHeader.name.toLowerCase()] ?? ""
      const a = Buffer.from(provided)
      const b = Buffer.from(options.webhookHeader.value)
      return a.length === b.length && timingSafeEqual(a, b)
    },
    extract: (raw) => JSON.parse(raw),
    // Tabby reuses the payment id across deliveries; the composite key changes per event.
    deliveryKey: (p) => `${p.id}:${p.status}:${p.captures?.length ?? 0}:${p.refunds?.length ?? 0}`,
  },
})
```

Notes on what the helper handles, vs what the adapter still has to do explicitly:

- **Money conversion is invisible.** `ctx.amount` is already `"100.50"` for AED, `"12.345"` for KWD. The framework computes that from the integer minor units Commerce Kit speaks internally. Amounts in payloads (e.g. `payment.amount`) are converted back to integer minor units before any `paymentTransaction` row is written.
- **Status mapper handles the `CLOSED` overload** by looking at `captures[]` and `refunds[]`. Returns one of the six outcome strings.
- **Pre-scoring rejection bypasses the status mapper** because it's a session-level signal that doesn't appear on the payment object. The method returns `{ outcome: "failed", ... }` directly.
- **`deliveryKey` is required** because Tabby's webhook `id` is the payment id, not a per-delivery event id. The composite key dedupes correctly: when a capture event arrives, the new `captures.length` makes the key differ from the prior authorize event.
- **Out-of-order webhook deliveries** are normal for Tabby. Core's `paymentTransaction` insert validation rejects out-of-order rows; Tabby retries up to 4 more times, so eventual consistency wins.

## Usage

```ts
// lib/commerce.ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { tabby } from "@commerce-kit/tabby"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [
    tabby({
      region: "ksa",
      secretKey: process.env.TABBY_SECRET_KEY!,
      merchantCode: process.env.TABBY_MERCHANT_CODE!,
      webhookHeader: {
        name: "X-Tabby-Token",
        value: process.env.TABBY_WEBHOOK_HEADER_VALUE!,
      },
    }),
  ],
})
```

Webhook mount: `POST /webhooks/payment/tabby`.

Multiple Tabby accounts (e.g. one per region) — `id` overrides at the call site:

```ts
payments: [
  tabby({ id: "tabby-ksa", region: "ksa", secretKey: env.TABBY_KSA_KEY, merchantCode: env.TABBY_KSA_CODE, webhookHeader: {...} }),
  tabby({ id: "tabby-uae", region: "uae", secretKey: env.TABBY_UAE_KEY, merchantCode: env.TABBY_UAE_CODE, webhookHeader: {...} }),
]
```

## Test plan

Money:
- AED 100.50 → `"100.50"`, round-tripping back to `10050` minor units.
- KWD 12.345 → `"12.345"`, round-tripping back to `12345` minor units.
- Rejects unsupported currency at the framework boundary before any network call.
- No floating-point math anywhere in the adapter source.

Authorize:
- Pre-scoring rejection: `configuration.products.installments.is_available === false` → `outcome: 'failed'` with `reason: "tabby:<code>"`. No transaction row inserted; order transitions to `failed`.
- Approved session with `web_url` → `outcome: 'requires_action'` with `paymentUrl` set. No transaction row.
- Approved session missing `web_url` throws `CommerceProviderError` rather than silently succeeding.
- `meta.order_id` is set on the create-session body so webhooks can round-trip back to the order.
- Region selection: `region: "ksa"` hits `api.tabby.sa`; `region: "uae"` hits `api.tabby.ai`.

Capture / refund / void:
- Capture sends `amount` as decimal string and `reference_id` for idempotency. On 200, `status(payload)` maps to `'captured'` and core appends one `CAPTURE` row.
- Refund against a `CLOSED` payment with prior `CAPTURE` rows works; refund against any other state returns 4xx → adapter throws → framework records `outcome: 'failed'`.
- Close on an `AUTHORIZED`-with-no-captures payment: `status(payload)` sees no captures, returns `'voided'`, core appends one `VOID` row.
- Close on a partially-captured payment: `status(payload)` sees captures, returns `'captured'` — core records this as a `CAPTURE` close-out, not a void.

Webhook:
- Correct header value passes; wrong value rejects with `CommerceWebhookError`.
- Lowercase `"authorized"` payload status normalized to uppercase by the status function.
- `"closed"` with `captures: [c1]` and empty `refunds[]` → `'captured'`.
- `"closed"` with `refunds: [r1]` → `'refunded'`.
- `"closed"` with empty `captures[]` and empty `refunds[]` → `'voided'`.
- `"rejected"` / `"expired"` → `'failed'`.
- Duplicate delivery (same composite `deliveryKey`) is dropped by core's webhook idempotency.
- Out-of-order delivery (capture event before authorize event) is retried by Tabby; the framework's transaction-insert validation rejects the temporary mismatch and core lets Tabby's retry handle it.

Configuration:
- Unsupported `region` value rejected at compile time (Standard Schema picklist).
- Missing `merchantCode` rejected at startup with a clear error.
