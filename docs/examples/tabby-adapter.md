# Tabby payment adapter (example)

> Companion to [`moyasar-adapter.md`](./moyasar-adapter.md). Wraps the [Tabby REST API](https://docs.tabby.ai/api-reference/overview) — a BNPL (buy-now-pay-later) provider for the Gulf region. The contract surface is identical to Moyasar's; what's interesting here is how a BNPL flow with provider-side credit scoring and major-unit string money maps onto the same `PaymentAdapter` shape.

## What's different from the Moyasar example

This adapter exists to show three things the Moyasar example doesn't exercise:

1. **Major-unit string money on the wire.** Tabby amounts are decimal strings (`"100.50"`), not integer minor units. The adapter converts at its boundary so Commerce Kit's "integer minor units only" invariant is preserved without leaking floats anywhere near the money path.
2. **Provider-side credit scoring at authorize time.** Tabby can reject a payment up front based on the buyer's BNPL eligibility. That maps to `outcome: 'failed'` with the rejection reason preserved — same outcome shape as a card decline, different mechanism.
3. **Hosted checkout (no card form).** There is no inline 3-D Secure or tokenized card. `authorize()` always returns either `requires_action` with a `web_url` or `failed` with a rejection. No `inline` flow.

Everything else — capability gating, the literal-typed `id`, the `paymentTransaction` model, webhook idempotency vs. transaction references, tenancy metadata round-trip — is identical to Moyasar's. Cross-reference rather than restate.

## Provider summary

| Concern | Tabby behavior |
|---|---|
| Base URL | `https://api.tabby.ai` |
| Auth | Bearer secret key in `Authorization`; `sk_test_*` for simulation, `sk_live_*` for production (prefix selects mode) |
| Money | **Decimal strings in major units** (`"100.50"`). Adapter converts to/from integer minor units. |
| Currencies | `SAR`, `AED`, `KWD`, `BHD`, `QAR` |
| Authorize | `POST /api/v2/checkout` — runs credit scoring, returns either an approved session with `configuration.available_products[].web_url`, or `status: "rejected"` with `rejection.code`. |
| Capture | `POST /api/v1/payments/{id}/captures` — supports partial. |
| Refund | `POST /api/v1/payments/{id}/refunds` — supports partial. |
| Void | `POST /api/v1/payments/{id}/close` — applies while `AUTHORIZED`. |
| Webhooks | HMAC-SHA256 in `x-tabby-signature`; payload `id` is the event id. |
| Provider statuses | `CREATED`, `AUTHORIZED`, `CLOSED`, `REJECTED`, `EXPIRED` |

## Capabilities

```ts
const tabbyCapabilities = {
  flow: "redirect",
  supportsCapture: true,
  supportsVoid: true,
  supportsRefund: true,
  supportsPartialRefund: true,
  supportsOffSession: false,         // BNPL requires the customer in-session
  voidableStates: ["AUTHORIZED"],
} as const satisfies PaymentCapabilities
```

Same shape as Moyasar; only `flow: "redirect"` is exercised — there is no inline form.

## Factory

```ts
// src/index.ts
export interface TabbyOptions {
  id?: string
  /** `sk_test_*` for simulation, `sk_live_*` for production. */
  secretKey: string
  /** Public key used by the storefront SDK; not used server-side but kept for parity. */
  publicKey?: string
  webhookSecret: string
  merchantCode: string               // Tabby account identifier, required on every checkout call
  baseUrl?: string                   // default "https://api.tabby.ai"
}

export function tabby<const Id extends string = "tabby">(
  options: TabbyOptions & { id?: Id },
): PaymentAdapter<Id> {
  return createTabbyAdapter({ id: "tabby" as Id, ...options })
}
```

## Money conversion

The interesting boundary on this adapter. Tabby wants `"100.50"`; Commerce Kit speaks `10050` (integer halalas / fils). Float math is banned — use string-based conversion that's exact.

```ts
// src/money.ts
const DECIMALS: Record<string, number> = {
  SAR: 2, AED: 2, QAR: 2,
  KWD: 3, BHD: 3,
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

/** integer minor units → "100.50" decimal string */
export function toTabbyAmount(minor: number, currency: string): string {
  const d = decimalsFor(currency)
  const sign = minor < 0 ? "-" : ""
  const abs = Math.abs(minor).toString().padStart(d + 1, "0")
  const cut = abs.length - d
  return `${sign}${abs.slice(0, cut)}.${abs.slice(cut)}`
}

/** "100.50" decimal string → integer minor units */
export function fromTabbyAmount(major: string, currency: string): number {
  const d = decimalsFor(currency)
  const match = /^(-?)(\d+)(?:\.(\d+))?$/.exec(major.trim())
  if (!match) {
    throw new CommerceProviderError(
      `Invalid Tabby amount: ${major}`,
      { providerId: "tabby", value: major },
    )
  }
  const [, sign, whole, frac = ""] = match
  const padded = (frac + "0".repeat(d)).slice(0, d)
  const minor = Number(whole) * 10 ** d + Number(padded)
  return sign ? -minor : minor
}
```

No `Number()` ever touches the cents portion; integer math only. Round-trip is lossless for valid inputs in every supported currency.

## Adapter implementation

```ts
// src/adapter.ts
import type {
  PaymentAdapter, AuthorizeContext, AuthorizeResult,
  CaptureResult, RefundResult, CancelResult,
} from "commerce-kit"
import { createClient } from "./client"
import { toTabbyAmount, fromTabbyAmount } from "./money"
import { mapTabbyOutcome } from "./map-outcome"
import { verifyTabbyWebhook } from "./webhook"

export function createTabbyAdapter<Id extends string>(
  options: ResolvedTabbyOptions<Id>,
): PaymentAdapter<Id> {
  const client = createClient(options)

  return {
    id: options.id,
    capabilities: tabbyCapabilities,

    async authorize(ctx: AuthorizeContext): Promise<AuthorizeResult> {
      const session = await client.post("/api/v2/checkout", {
        idempotencyKey: ctx.idempotencyKey,
        body: {
          payment: {
            amount: toTabbyAmount(ctx.amount, ctx.currency),
            currency: ctx.currency,
            buyer: requireBuyer(ctx),         // email, phone, name — Tabby requires these
            shipping_address: ctx.shippingAddress,
            order: {
              reference_id: ctx.orderId,
              items: ctx.items.map(toTabbyItem),
            },
            meta: {
              order_id: ctx.orderId,
              merchant_id: ctx.merchantId ?? null,
              branch_id: ctx.branchId ?? null,
            },
          },
          merchant_code: options.merchantCode,
          lang: ctx.locale ?? "en",
          merchant_urls: {
            success: ctx.returnUrl,
            cancel: ctx.cancelUrl,
            failure: ctx.failureUrl ?? ctx.returnUrl,
          },
        },
      })

      // Rejection at scoring time — no money moved, no transaction row.
      if (session.status === "rejected") {
        return {
          outcome: "failed",
          providerReference: session.payment.id,
          reason: `tabby:${session.rejection?.code ?? "rejected"}`,
        }
      }

      // Approved — customer must complete on Tabby's hosted page.
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
      const captured = await client.post(`/api/v1/payments/${providerReference}/captures`, {
        body: { amount: toTabbyAmount(amount, captureCurrencyFor(providerReference)) },
      })
      return mapTabbyOutcome(captured) === "captured"
        ? { outcome: "captured", providerReference, amount: fromTabbyAmount(captured.amount, captured.currency) }
        : { outcome: "failed", reason: captured.error?.message ?? "capture failed" }
    },

    async refund(providerReference, amount, reason): Promise<RefundResult> {
      const refunded = await client.post(`/api/v1/payments/${providerReference}/refunds`, {
        body: {
          amount: toTabbyAmount(amount, captureCurrencyFor(providerReference)),
          reason: reason ?? "merchant_refund",
        },
      })
      return mapTabbyOutcome(refunded) === "refunded"
        ? { outcome: "refunded", providerReference, amount: fromTabbyAmount(refunded.amount, refunded.currency) }
        : { outcome: "failed", reason: reason ?? "refund failed" }
    },

    async cancel(providerReference): Promise<CancelResult> {
      const closed = await client.post(`/api/v1/payments/${providerReference}/close`)
      return mapTabbyOutcome(closed) === "voided"
        ? { outcome: "voided", providerReference }
        : { outcome: "failed", reason: closed.error?.message ?? "void failed" }
    },

    async verifyWebhook(payload, signature) {
      return verifyTabbyWebhook(payload, signature, options.webhookSecret)
    },
  }
}
```

Two things to call out:

- The currency for capture/refund isn't passed by core — it's recorded on the original `paymentTransaction.currency` when the AUTHORIZE row was written. `captureCurrencyFor(providerReference)` is a small lookup against the ledger. (Alternative: have core thread the currency through; that's a future contract tweak.)
- Tabby's checkout response lists multiple installment products under `configuration.available_products`. The example picks the first; a real adapter should expose the product list via `AuthorizeResult.options` so the storefront can let the customer pick (4-installment vs pay-in-30-days, etc.). Out of scope here.

## Outcome mapping

```ts
// src/map-outcome.ts
export type TabbyOutcome =
  | "authorized" | "captured" | "refunded" | "voided" | "requires_action" | "failed"

export function mapTabbyOutcome(p: { status: string }): TabbyOutcome {
  switch (p.status) {
    case "CREATED":    return "requires_action"  // session approved, awaiting customer redirect
    case "AUTHORIZED": return "authorized"
    case "CLOSED":
      // CLOSED is overloaded in Tabby's API — it's used for both captured and voided.
      // Disambiguate by whether captures exist. Adapter doesn't see that detail on
      // every endpoint, so capture()/cancel() return the right outcome explicitly
      // and only the webhook path uses this branch.
      return "captured"
    case "REJECTED":   return "failed"
    case "EXPIRED":    return "failed"
    default:
      throw new CommerceProviderError(
        `Unknown Tabby status: ${p.status}`,
        { providerId: "tabby", providerStatus: p.status },
      )
  }
}
```

`CLOSED` is the tricky one — Tabby reuses it for capture-completion and void. The webhook event type (`payment.captured` vs `payment.closed`) disambiguates; the adapter's `capture()` and `cancel()` methods don't rely on `mapTabbyOutcome` at all and return their outcome directly.

## Webhook verification

```ts
// src/webhook.ts
import { createHmac, timingSafeEqual } from "node:crypto"
import { mapTabbyOutcome } from "./map-outcome"

export async function verifyTabbyWebhook(payload: unknown, signature: string, secret: string) {
  const raw = typeof payload === "string" ? payload : JSON.stringify(payload)
  const expected = createHmac("sha256", secret).update(raw).digest("hex")
  const given = Buffer.from(signature ?? "", "hex")
  const want = Buffer.from(expected, "hex")
  if (given.length !== want.length || !timingSafeEqual(given, want)) {
    throw new CommerceWebhookError("Invalid Tabby webhook signature")
  }
  const event = JSON.parse(raw) as TabbyWebhookPayload

  return {
    eventId: event.id,
    providerReference: event.payment.id,
    orderId: event.payment.meta?.order_id,
    outcome: outcomeFromEventType(event.event) ?? mapTabbyOutcome(event.payment),
    amount: fromTabbyAmount(event.payment.amount, event.payment.currency),
    currency: event.payment.currency,
    raw: event,
  }
}

function outcomeFromEventType(t: string): TabbyOutcome | null {
  switch (t) {
    case "payment.authorized": return "authorized"
    case "payment.captured":   return "captured"
    case "payment.refunded":   return "refunded"
    case "payment.closed":     return "voided"    // Tabby's "close" = void from the merchant
    case "payment.expired":
    case "payment.rejected":   return "failed"
    default: return null
  }
}
```

The event-type-first / status-fallback order resolves the `CLOSED` ambiguity cleanly — `payment.captured` and `payment.closed` are distinct events even though they share a state.

## Usage

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { tabby } from "@commerce-kit/tabby"

export const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [
    tabby({
      secretKey: process.env.TABBY_SECRET_KEY!,        // sk_test_… or sk_live_…
      publicKey: process.env.TABBY_PUBLIC_KEY,
      webhookSecret: process.env.TABBY_WEBHOOK_SECRET!,
      merchantCode: process.env.TABBY_MERCHANT_CODE!,
    }),
  ],
})
```

Side-by-side with Moyasar — both adapters expose the same `commerce.payments.*` surface, and checkout selects which one to use:

```ts
payments: [
  moyasar({ secretKey: env.MOYASAR_SECRET_KEY, webhookSecret: env.MOYASAR_WEBHOOK_SECRET }),
  tabby({   secretKey: env.TABBY_SECRET_KEY,   webhookSecret: env.TABBY_WEBHOOK_SECRET, merchantCode: env.TABBY_MERCHANT_CODE }),
]

await commerce.orders.checkout({
  items: cart.items,
  payment: { adapterId: "tabby", providerInput: { buyer, shippingAddress } },
})
```

Webhook mount: `POST /webhooks/payment/tabby`.

## Test plan

Money:
- `toTabbyAmount(10050, "SAR") === "100.50"` and `fromTabbyAmount("100.50", "SAR") === 10050`.
- Round-trip for KWD/BHD (3-decimal): `12345` ↔ `"12.345"`.
- Rejects unsupported currencies before any network call.
- No `Number()` on the cents portion; verified by passing `"99999999999.99"` (above safe integer if naively multiplied as a float) and checking round-trip.

Authorize:
- Tabby returns `status: "rejected"` → `outcome: 'failed'` with `reason` carrying the Tabby rejection code; no transaction row inserted; `order.status` transitions to `failed`.
- Tabby returns approved session with a `web_url` → `outcome: 'requires_action'`; no transaction row; storefront receives the `web_url` synchronously.
- Approved session missing `web_url` raises `CommerceProviderError` rather than silently succeeding.
- `meta.order_id` is forwarded for webhook round-trip; tenancy ids are included when set.

Capture / refund / void:
- Capture appends one `CAPTURE` row with amount converted from Tabby's response string.
- Refund against an authorized-but-never-captured payment routes through `cancel()` (Tabby's `/close`) per the void/refund cascade in [50-adapter-system.md](../architecture/50-adapter-system.md).
- Partial refund appends one `REFUND` row; core's `Σrefund < Σcapture` rule projects `PARTIALLY_REFUNDED`.

Webhook:
- `payment.captured` event ⇒ `outcome: 'captured'`, core appends `CAPTURE` row.
- `payment.closed` event ⇒ `outcome: 'voided'`, core appends `VOID` row — *not* misread as captured despite the shared `CLOSED` status.
- Invalid signature rejects with `CommerceWebhookError`.
- Replay of the same `event.id` is a no-op at core's webhook idempotency layer.
