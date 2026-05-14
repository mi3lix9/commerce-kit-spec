# Tabby payment adapter (example)

> Companion to [`moyasar-adapter.md`](./moyasar-adapter.md). Wraps the [Tabby API](https://docs.tabby.ai/api-reference/overview) — a BNPL (buy-now-pay-later) provider for the Gulf region. This example reconciles against the real Tabby docs (crawled 2026-05-14); cross-check before shipping in case the contract has moved.
>
> Source pages used: [overview](https://docs.tabby.ai/api-reference/overview), [create-a-session](https://docs.tabby.ai/api-reference/checkout/create-a-session), [payment-statuses](https://docs.tabby.ai/pay-in-4-custom-integration/payment-statuses), [payment-processing](https://docs.tabby.ai/pay-in-4-custom-integration/payment-processing), [webhooks](https://docs.tabby.ai/pay-in-4-custom-integration/webhooks), [register-a-webhook](https://docs.tabby.ai/api-reference/webhooks/register-a-webhook), [technical-requirements](https://docs.tabby.ai/introduction/technical-requirements).

## What's different from the Moyasar example

Five concrete differences worth exercising in this example:

1. **Region-specific base URLs.** Tabby uses `api.tabby.ai` for UAE/KW and `api.tabby.sa` for KSA. Same payloads on both. The adapter picks per-instance from `region`, not from currency.
2. **Decimal-string money on the wire.** Amounts are strings like `"100.00"` — up to 2 decimals for AED/SAR, up to 3 for KWD. (Tabby's docs call this "minor units" but the example is major-unit; we treat it as a decimal-with-fixed-scale and convert via integer math so neither Commerce Kit nor the wire ever sees a float.)
3. **Pre-scoring at session creation.** Tabby runs a credit check at `POST /api/v2/checkout`. The session succeeds with `200` even when the customer is rejected — you detect it via `configuration.products.installments.is_available === false` plus a `rejection_reason`. Maps to `outcome: 'failed'` without ever showing the customer a redirect page.
4. **Webhook authentication is a configurable header, not HMAC.** At webhook registration you supply `{ header: { title, value } }`. Tabby echoes that header on every delivery; verification is a constant-time string compare of the value.
5. **Webhooks have no `event` field.** The same payload shape is sent for authorize, capture, close, refund, expire. You derive the event from `status` + the presence of new entries in `captures[]` / `refunds[]`. Order is not guaranteed and duplicates are possible.

Other contract details (factory `id` generic, `paymentTransaction` ledger, capability gating, tenancy metadata round-trip) are identical to the Moyasar example — cross-reference rather than restate.

## Provider summary

| Concern | Tabby behavior |
|---|---|
| Base URLs | `https://api.tabby.ai` (UAE/KW) and `https://api.tabby.sa` (KSA). Identical schemas, region routing is by merchant. |
| Auth | `Authorization: Bearer <secret_key>`; test vs live is selected by the key. |
| Money | Decimal strings: `"100.00"` for AED/SAR, `"100.000"` for KWD. |
| Currencies | `AED`, `SAR`, `KWD` (also documented: `BHD`, `QAR` for some plans). |
| Create session | `POST /api/v2/checkout` — runs pre-scoring. Returns `{ id, payment, configuration, status }`. `configuration.available_products.installments[].web_url` is the redirect URL when approved. |
| Capture | `POST /api/v2/payments/{id}/captures` — body `{ amount, reference_id }`. `reference_id` is the idempotency key. |
| Refund | `POST /api/v2/payments/{id}/refunds` — body `{ amount, reference_id, reason }`. Multiple partial refunds allowed; only `CLOSED` payments are refundable. |
| Close | `POST /api/v2/payments/{id}/close` — empty body. Used to cancel an `AUTHORIZED` payment (no capture) or to close the remaining balance after partial captures. |
| Retrieve | `GET /api/v2/payments/{id}` — returns the full payment object with **uppercase** `status`. |
| Payment statuses (Retrieve) | `CREATED`, `AUTHORIZED`, `CLOSED`, `REJECTED`, `EXPIRED` |
| Webhook statuses (delivery) | Same set but **lowercase**: `authorized`, `closed`, `rejected`, `expired`. |
| Webhook auth | Custom header configured at registration (e.g. `X-My-Tabby-Token: <secret>`). No HMAC. |
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
| `Close` after partial capture (refunds remainder) | `CLOSED` | `CAPTURED` |
| `Close` after authorize, no capture | `CLOSED` | `CANCELLED` |
| Refund(s) after capture | `CLOSED` | `REFUNDED` / `PARTIALLY REFUNDED` |

The API status alone cannot tell you whether a `CLOSED` payment was captured, cancelled, or refunded. The adapter disambiguates by looking at `captures[]` and `refunds[]` in the payment object:

- `captures.length > 0` and `refunds.length === 0` ⇒ captured.
- `refunds.length > 0` ⇒ refunded (full if `Σrefund == Σcapture`, partial otherwise — but Commerce Kit derives that itself from `paymentTransaction`, the adapter just emits `'refunded'`).
- `captures.length === 0` ⇒ voided (close-after-authorize).

## Capabilities

```ts
const tabbyCapabilities = {
  flow: "redirect",
  supportsCapture: true,
  supportsVoid: true,
  supportsRefund: true,
  supportsPartialRefund: true,
  supportsOffSession: false,
  voidableStates: ["AUTHORIZED"],
} as const satisfies PaymentCapabilities
```

## Factory

```ts
// src/index.ts
export interface TabbyOptions {
  /** Default: "tabby". Override for multiple Tabby accounts. */
  id?: string

  /** Secret API key from the Merchant Dashboard. Test or live mode is selected by the key itself. */
  secretKey: string

  /** Public key, used by the storefront SDK. Not consumed server-side but kept for parity. */
  publicKey?: string

  /** Merchant code from Tabby — required on every checkout session. */
  merchantCode: string

  /** Region picks the base URL. Same payloads across regions. */
  region: "uae" | "ksa"

  /**
   * Header name + value configured when registering the webhook with Tabby.
   * Tabby echoes this header on every webhook delivery; verification is a
   * constant-time string compare. There is no HMAC.
   */
  webhookHeader: { name: string; value: string }
}

export function tabby<const Id extends string = "tabby">(
  options: TabbyOptions & { id?: Id },
): PaymentAdapter<Id> {
  return createTabbyAdapter({ id: "tabby" as Id, ...options })
}

const BASE_URLS = {
  uae: "https://api.tabby.ai",
  ksa: "https://api.tabby.sa",
} as const
```

## Money conversion

Tabby amounts are decimal strings. Commerce Kit speaks integer minor units. Conversion runs string-based on the digits — no `Number()` on anything but the integer part — so float precision never enters the money path.

```ts
// src/money.ts
const DECIMALS: Record<string, number> = {
  SAR: 2, AED: 2, QAR: 2, BHD: 3, KWD: 3,
}

function decimalsFor(currency: string): number {
  const d = DECIMALS[currency]
  if (d === undefined) {
    throw new CommerceProviderError(
      `Tabby does not support currency ${currency}`,
      { providerId: "tabby", currency },
    )
  }
  return d
}

/** Integer minor units → Tabby's decimal string. `10050, "AED"` → `"100.50"`. */
export function toTabbyAmount(minor: number, currency: string): string {
  const d = decimalsFor(currency)
  if (!Number.isInteger(minor)) {
    throw new CommerceProviderError(`amount must be integer minor units, got ${minor}`,
      { providerId: "tabby" })
  }
  const sign = minor < 0 ? "-" : ""
  const abs = Math.abs(minor).toString().padStart(d + 1, "0")
  const cut = abs.length - d
  return `${sign}${abs.slice(0, cut)}.${abs.slice(cut)}`
}

/** Tabby's decimal string → integer minor units. `"100.50", "AED"` → `10050`. */
export function fromTabbyAmount(major: string, currency: string): number {
  const d = decimalsFor(currency)
  const m = /^(-?)(\d+)(?:\.(\d+))?$/.exec(major.trim())
  if (!m) {
    throw new CommerceProviderError(`Invalid Tabby amount: ${major}`,
      { providerId: "tabby", value: major })
  }
  const [, sign, whole, frac = ""] = m
  if (frac.length > d) {
    throw new CommerceProviderError(
      `Amount "${major}" has more decimals than ${currency} allows (${d})`,
      { providerId: "tabby" })
  }
  const padded = (frac + "0".repeat(d)).slice(0, d)
  const minor = Number(whole) * 10 ** d + Number(padded)
  return sign ? -minor : minor
}
```

## Adapter

```ts
// src/adapter.ts
import type {
  PaymentAdapter, AuthorizeContext, AuthorizeResult,
  CaptureResult, RefundResult, CancelResult,
} from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { createClient } from "./client"
import { toTabbyAmount, fromTabbyAmount } from "./money"
import { verifyTabbyWebhook } from "./webhook"
import { tabbyCapabilities, BASE_URLS } from "./constants"

export function createTabbyAdapter<Id extends string>(
  options: ResolvedTabbyOptions<Id>,
): PaymentAdapter<Id> {
  const client = createClient({ baseUrl: BASE_URLS[options.region], secretKey: options.secretKey })

  return {
    id: options.id,
    capabilities: tabbyCapabilities,

    async authorize(ctx: AuthorizeContext): Promise<AuthorizeResult> {
      const session = await client.post("/api/v2/checkout", {
        body: {
          merchant_code: options.merchantCode,
          lang: ctx.locale ?? "en",
          merchant_urls: {
            success: ctx.returnUrl,
            cancel: ctx.cancelUrl ?? ctx.returnUrl,
            failure: ctx.failureUrl ?? ctx.returnUrl,
          },
          payment: {
            amount: toTabbyAmount(ctx.amount, ctx.currency),
            currency: ctx.currency,
            description: ctx.description,
            buyer: requireBuyer(ctx),
            shipping_address: ctx.shippingAddress,
            order: {
              reference_id: ctx.orderId,
              tax_amount: toTabbyAmount(ctx.taxAmount ?? 0, ctx.currency),
              shipping_amount: toTabbyAmount(ctx.shippingAmount ?? 0, ctx.currency),
              discount_amount: toTabbyAmount(ctx.discountAmount ?? 0, ctx.currency),
              items: ctx.items.map((i) => toTabbyItem(i, ctx.currency)),
            },
            buyer_history: ctx.buyerHistory,    // optional but improves approval rates
            order_history: ctx.orderHistory,    // optional but improves approval rates
            meta: {
              order_id: ctx.orderId,            // round-trips on webhooks so core can resolve
              merchant_id: ctx.merchantId ?? null, // the order without a persisted reference
              branch_id: ctx.branchId ?? null,
            },
          },
        },
      })

      // Pre-scoring rejection: session succeeds, but the installments product is unavailable.
      const installments = session.configuration?.products?.installments
      if (installments && installments.is_available === false) {
        return {
          outcome: "failed",
          providerReference: session.payment.id,
          reason: `tabby:${installments.rejection_reason ?? "rejected"}`,
        }
      }

      // Approved — redirect the customer to the hosted page.
      const webUrl = session.configuration?.available_products?.installments?.[0]?.web_url
      if (!webUrl) {
        throw new CommerceProviderError(
          "Tabby returned an approved session without a web_url",
          { providerId: "tabby", sessionId: session.id },
        )
      }
      return {
        outcome: "requires_action",
        providerReference: session.payment.id,
        paymentUrl: webUrl,
      }
    },

    async capture(providerReference, amount): Promise<CaptureResult> {
      // Currency for the wire-format conversion is recorded on the original
      // paymentTransaction row. The adapter looks it up via core's read helper;
      // see "Money currency lookup" below.
      const currency = await currencyForPayment(providerReference)
      const payment = await client.post(
        `/api/v2/payments/${providerReference}/captures`,
        {
          body: {
            amount: toTabbyAmount(amount, currency),
            reference_id: `cap-${providerReference}-${amount}`,   // idempotency key
          },
        },
      )
      return payment.status === "CLOSED"
        ? { outcome: "captured", providerReference, amount: fromTabbyAmount(payment.amount, payment.currency) }
        : { outcome: "failed",   reason: `unexpected status ${payment.status} after capture` }
    },

    async refund(providerReference, amount, reason): Promise<RefundResult> {
      const currency = await currencyForPayment(providerReference)
      const payment = await client.post(
        `/api/v2/payments/${providerReference}/refunds`,
        {
          body: {
            amount: toTabbyAmount(amount, currency),
            reference_id: `ref-${providerReference}-${amount}-${Date.now()}`,
            reason: reason ?? "merchant_refund",
          },
        },
      )
      // After a successful refund Tabby still reports status: "CLOSED" — the new
      // refund row is in payment.refunds[]. We trust the 200 response.
      return { outcome: "refunded", providerReference, amount }
    },

    async cancel(providerReference): Promise<CancelResult> {
      const payment = await client.post(`/api/v2/payments/${providerReference}/close`)
      // Close after AUTHORIZED with no captures = void. Close after capture =
      // closing remainder (not a void). The adapter only calls cancel() per the
      // orders.refund cascade, which already verified the payment is in voidableStates.
      if (payment.status !== "CLOSED" || (payment.captures?.length ?? 0) > 0) {
        return { outcome: "failed", reason: "close did not result in a void" }
      }
      return { outcome: "voided", providerReference }
    },

    async verifyWebhook(payload, signature, headers) {
      return verifyTabbyWebhook({ payload, headers, expected: options.webhookHeader })
    },
  }
}
```

### Money currency lookup

`capture()` / `refund()` need the currency to format the outgoing amount, but the core contract passes only `providerReference` and an integer minor-unit `amount`. The currency lives on the original `paymentTransaction.AUTHORIZE` row written when `authorize()` returned.

Two options:

- **Lookup at the seam** — add `commerce.payments._currencyFor(providerReference)` to the adapter context. Cleanest; minor core surface addition.
- **Thread currency through** — change the contract so `capture(providerReference, amount, currency)`. Bigger surface change but more explicit.

This example assumes the first; track the contract decision in the adapter system doc.

## Webhook verification

```ts
// src/webhook.ts
import { timingSafeEqual } from "node:crypto"
import { CommerceWebhookError } from "commerce-kit/errors"
import { fromTabbyAmount } from "./money"

interface VerifyArgs {
  payload: unknown
  headers: Record<string, string | undefined>
  expected: { name: string; value: string }
}

export async function verifyTabbyWebhook({ payload, headers, expected }: VerifyArgs) {
  const provided = headers[expected.name.toLowerCase()] ?? ""
  const a = Buffer.from(provided)
  const b = Buffer.from(expected.value)
  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    throw new CommerceWebhookError("Invalid Tabby webhook header")
  }

  const event = typeof payload === "string" ? JSON.parse(payload) : payload as TabbyWebhookPayload

  return {
    eventId: event.id,                          // Tabby reuses the payment id; combine with status
    deliveryKey: `${event.id}:${event.status}:${event.captures?.length ?? 0}:${event.refunds?.length ?? 0}`,
    providerReference: event.id,
    orderId: event.meta?.order_id,
    outcome: deriveOutcome(event),
    amount: fromTabbyAmount(event.amount, event.currency),
    currency: event.currency,
    raw: event,
  }
}

/**
 * Tabby webhooks have no event field. Derive the event from the payment shape.
 * Webhook status values are lowercase; Retrieve responses use uppercase. We
 * uppercase here so the rest of the adapter sees one form.
 */
function deriveOutcome(p: TabbyWebhookPayload): TabbyOutcome {
  const status = p.status.toUpperCase()
  switch (status) {
    case "AUTHORIZED":
      // Authorize, capture-while-authorized, or update — all arrive with this
      // status. captures.length > 0 means a capture event; otherwise it's the
      // initial authorize. Core's transaction-uniqueness rules de-dup either way.
      return (p.captures?.length ?? 0) > 0 ? "captured" : "authorized"
    case "CLOSED":
      // Three cases share this status. Disambiguate via the arrays.
      if ((p.refunds?.length ?? 0) > 0)   return "refunded"
      if ((p.captures?.length ?? 0) > 0)  return "captured"
      return "voided"
    case "REJECTED":
    case "EXPIRED":
      return "failed"
    default:
      throw new CommerceWebhookError(`Unknown Tabby webhook status: ${p.status}`)
  }
}
```

Notes core care about:

- **`deliveryKey` for idempotency.** Tabby's webhook `id` is the payment id, which repeats across deliveries. The composite key (`paymentId:status:captures:refunds`) is what changes per event and is what core's webhook idempotency layer should de-dup on.
- **Out-of-order delivery.** Tabby explicitly does not guarantee order. Core's `paymentTransaction` insert validation (see [50-adapter-system.md](../architecture/50-adapter-system.md)) rejects out-of-order events — e.g. a `captured` arriving before its `authorized`. Treat that as a soft failure and retry the lookup; eventual consistency wins because Tabby retries up to 4 times.
- **Allowlist by Tabby's webhook IP range** at the framework adapter layer for defense in depth. The header check is the primary verification.

## Usage

```ts
// lib/commerce.ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { tabby } from "@commerce-kit/tabby"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [
    tabby({
      region: "ksa",                              // or "uae"
      secretKey: process.env.TABBY_SECRET_KEY!,
      publicKey: process.env.TABBY_PUBLIC_KEY,
      merchantCode: process.env.TABBY_MERCHANT_CODE!,
      webhookHeader: {
        name: "X-Tabby-Token",
        value: process.env.TABBY_WEBHOOK_HEADER_VALUE!,
      },
    }),
  ],
})
```

Webhook mount: `POST /webhooks/payment/tabby`. The framework adapter forwards request headers to `verifyWebhook` so the configured header name can be inspected.

Multiple Tabby accounts (e.g. one per region):

```ts
payments: [
  tabby({ id: "tabby-ksa", region: "ksa", secretKey: env.TABBY_KSA_KEY, merchantCode: env.TABBY_KSA_CODE, webhookHeader: {...} }),
  tabby({ id: "tabby-uae", region: "uae", secretKey: env.TABBY_UAE_KEY, merchantCode: env.TABBY_UAE_CODE, webhookHeader: {...} }),
]
```

## Test plan

Money:
- `toTabbyAmount(10050, "SAR") === "100.50"`.
- `fromTabbyAmount("100.50", "SAR") === 10050`.
- 3-decimal currency: `12345` ↔ `"12.345"` for KWD/BHD.
- Rejects amount with more decimals than the currency allows (`"100.123"` for AED → throws).
- Rejects unsupported currency before any network call.
- Non-integer minor units throw — defensive guard against accidental float in.

Authorize:
- Pre-scoring rejection: `configuration.products.installments.is_available === false` → `outcome: 'failed'` with `reason: "tabby:<code>"`. No transaction row inserted; order transitions to `failed`.
- Approved session with `web_url` → `outcome: 'requires_action'` with `paymentUrl` set. No transaction row.
- Approved session missing `web_url` raises `CommerceProviderError` rather than silently succeeding.
- `meta.order_id` is set on the create-session body so webhooks can round-trip back to the order.
- Region selection: `region: "ksa"` hits `api.tabby.sa`; `region: "uae"` hits `api.tabby.ai`.

Capture / refund / void:
- Capture sends `amount` as decimal string and `reference_id` for idempotency. On 200 with `status: "CLOSED"`, returns `outcome: 'captured'`.
- Partial capture on a payment with `status: "AUTHORIZED"` (Tabby keeps the payment in AUTHORIZED until full capture or close).
- Refund body includes `reference_id` and `reason`. Refunds against a `CLOSED` payment work; against any other state, Tabby returns 4xx and the adapter returns `outcome: 'failed'`.
- Close on an AUTHORIZED payment with no captures returns `outcome: 'voided'`.
- Close on a partially-captured payment is **not** a void — adapter returns `outcome: 'failed'` so core doesn't mis-record the result. (Core's void path should only run on the AUTHORIZED-no-captures case anyway, but defense in depth.)

Webhook:
- Correct header value passes; wrong value rejects with `CommerceWebhookError`.
- Lowercase `"authorized"` payload status is recognized.
- `"closed"` with `captures: [c1]` and empty `refunds[]` → `outcome: 'captured'`.
- `"closed"` with `refunds: [r1]` → `outcome: 'refunded'`.
- `"closed"` with empty `captures[]` → `outcome: 'voided'`.
- `"rejected"` / `"expired"` → `outcome: 'failed'`.
- Duplicate delivery (same `deliveryKey`) is dropped by core's webhook idempotency.
- Out-of-order delivery (capture-event before authorize-event) is retried by Tabby; the adapter must not throw on the temporary mismatch.

Configuration:
- Unsupported `region` value rejected at compile time (literal union).
- Missing `merchantCode` rejected at startup with a clear error.
