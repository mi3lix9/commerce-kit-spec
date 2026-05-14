# Moyasar payment adapter (example)

> Illustrative reference for authoring a Commerce Kit payment adapter against a real provider. Wraps the [Moyasar REST API](https://docs.moyasar.com/api/api-introduction). Signatures track [50-adapter-system.md](../architecture/50-adapter-system.md); refine against the canonical contracts before shipping.

## What this example shows

- How a provider package conforms to the `PaymentAdapter` interface.
- How `capabilities` map to Moyasar's actual behavior, so the SDK surface narrows correctly at compile time.
- How redirect flow (`source.type: "creditcard"` with 3-D Secure) and inline flow (Moyasar.js) are both expressed through a single adapter.
- How the adapter returns a neutral **outcome** (`'authorized' | 'captured' | 'refunded' | 'voided' | 'requires_action' | 'failed'`) and lets core decide whether to append a `paymentTransaction` row or transition `order.status`.
- How `verifyWebhook` validates Moyasar's webhook signature and round-trips `order_id` via provider metadata (the order has no persisted `paymentReference`).
- How idempotency, integer minor units (halalas), and webhook event IDs are kept distinct from transaction references.

This is a **single-adapter example**. The same adapter works unchanged in single-store, multi-merchant, and marketplace (`tenancy.checkout: "split"`) configurations — the adapter is tenancy-agnostic; Commerce Kit owns tenancy routing.

## Provider summary

Moyasar is a Saudi Arabia–based PSP. Key facts that shape the adapter:

| Concern | Moyasar behavior |
|---|---|
| Base URL | `https://api.moyasar.com/v1` |
| Auth | HTTP Basic; secret key as username, empty password |
| Money | Integer minor units (halalas for SAR). Matches Commerce Kit's contract. |
| Currencies | `SAR`, `USD`, `AED`, `EGP`, `KWD`, `BHD`, `QAR`, `OMR`, `JOD`. |
| Sources | `creditcard`, `applepay`, `stcpay`, `samsungpay`. |
| Auth/capture | Optional via `description` + `auto_capture: false` on the payment create call. Capture via `POST /payments/:id/capture`. |
| Void | `POST /payments/:id/void` while `authorized`. |
| Refund | `POST /payments/:id/refund`; supports partial. |
| Webhooks | HMAC-SHA256 signature in `X-Moyasar-Signature`; event ids in payload `id`. |
| Statuses | `initiated`, `paid`, `failed`, `authorized`, `captured`, `refunded`, `voided`. |

## Package shape

```text
packages/moyasar/
├─ src/
│  ├─ index.ts            ← exports moyasar() factory
│  ├─ adapter.ts          ← PaymentAdapter impl
│  ├─ client.ts           ← thin fetch wrapper (Basic auth, retries, idempotency)
│  ├─ map-outcome.ts      ← Moyasar status → neutral outcome union
│  ├─ webhook.ts          ← signature verification + event mapping
│  └─ types.ts            ← Moyasar payload types
├─ test/
│  └─ adapter.test.ts
└─ package.json           ← "@commerce-kit/moyasar"
```

## Capabilities

Moyasar supports manual capture, void, and full/partial refund. It is not an off-session provider in the Commerce Kit sense (no SDK-managed saved-customer charge), so `supportsOffSession` is `false`.

```ts
const moyasarCapabilities = {
  flow: "redirect",
  supportsCapture: true,
  supportsVoid: true,
  supportsRefund: true,
  supportsPartialRefund: true,
  supportsOffSession: false,
  voidableStates: ["AUTHORIZED"],
} as const satisfies PaymentCapabilities
```

The `as const satisfies` pattern preserves literal types so the SDK can narrow against this adapter's capabilities. See [60-type-inference.md](../architecture/60-type-inference.md).

The same factory could expose `flow: "inline"` behind a config flag (`{ inline: true }`) and return `inlineSecret` from `authorize` instead of `paymentUrl`. Keep that as a follow-up — start with redirect.

## Factory shape

```ts
// src/index.ts
import type { PaymentAdapter } from "commerce-kit"
import { createMoyasarAdapter } from "./adapter"

export interface MoyasarOptions {
  /** Default: "moyasar". Override when registering multiple Moyasar accounts. */
  id?: string
  /** `sk_test_*` for simulation, `sk_live_*` for production — the prefix selects the mode. */
  secretKey: string
  webhookSecret: string
  /** Default: "https://api.moyasar.com/v1" */
  baseUrl?: string
  /** Default: ["creditcard", "applepay", "stcpay"] */
  sources?: Array<"creditcard" | "applepay" | "stcpay" | "samsungpay">
  /** Default: false. When true, payments are captured at authorize time. */
  autoCapture?: boolean
}

export function moyasar<const Id extends string = "moyasar">(
  options: MoyasarOptions & { id?: Id },
): PaymentAdapter<Id> {
  return createMoyasarAdapter({ id: "moyasar" as Id, ...options })
}
```

The `const Id` generic lets `createCommerce()` infer the literal id so `order.payment.adapter` is typed as `"moyasar" | ...` rather than `string`. Duplicate ids in the `payments: []` slot are caught at compile time per [50-adapter-system.md](../architecture/50-adapter-system.md).

## Adapter implementation

```ts
// src/adapter.ts
import type {
  PaymentAdapter,
  AuthorizeContext,
  AuthorizeResult,
  CaptureResult,
  RefundResult,
  CancelResult,
} from "commerce-kit"
import { CommerceProviderError } from "commerce-kit/errors"
import { createClient } from "./client"
import { mapMoyasarOutcome } from "./map-outcome"
import { verifyMoyasarWebhook } from "./webhook"

export function createMoyasarAdapter<Id extends string>(
  options: ResolvedMoyasarOptions<Id>,
): PaymentAdapter<Id> {
  const client = createClient(options)

  return {
    id: options.id,

    capabilities: moyasarCapabilities,

    async authorize(ctx: AuthorizeContext): Promise<AuthorizeResult> {
      assertSupportedCurrency(ctx.currency)

      // ctx.amount is already integer minor units (halalas for SAR).
      // ctx.idempotencyKey is provided by core; use it as the Idempotency-Key header.
      const payment = await client.post("/payments", {
        idempotencyKey: ctx.idempotencyKey,
        body: {
          amount: ctx.amount,
          currency: ctx.currency,
          description: ctx.description ?? `Order ${ctx.orderId}`,
          callback_url: ctx.returnUrl,
          source: {
            type: "creditcard",
            // Token from Moyasar.js on the storefront, passed through ctx.providerInput.
            token: requireToken(ctx),
          },
          metadata: {
            order_id: ctx.orderId,             // round-trips back via webhooks so core
            merchant_id: ctx.merchantId ?? null, // can resolve the order without a
            branch_id: ctx.branchId ?? null,     // persisted paymentReference field.
          },
          auto_capture: options.autoCapture ?? false,
        },
      })

      const outcome = mapMoyasarOutcome(payment)
      switch (outcome) {
        case "authorized":
        case "captured":
          return { outcome, providerReference: payment.id, amount: payment.amount }
        case "requires_action":
          // No transaction is persisted yet. paymentUrl is transient — core returns
          // it to the caller; the application caches it in Redis if it needs to
          // resume the session.
          return {
            outcome: "requires_action",
            providerReference: payment.id,
            paymentUrl: payment.source?.transaction_url,
          }
        case "failed":
          return {
            outcome: "failed",
            providerReference: payment.id,
            reason: payment.source?.message ?? "declined",
          }
      }
    },

    async capture(providerReference, amount): Promise<CaptureResult> {
      const payment = await client.post(
        `/payments/${providerReference}/capture`,
        { body: { amount } },
      )
      return mapMoyasarOutcome(payment) === "captured"
        ? { outcome: "captured", providerReference: payment.id, amount: payment.captured ?? amount }
        : { outcome: "failed",   reason: payment.source?.message ?? "capture failed" }
    },

    async refund(providerReference, amount, reason): Promise<RefundResult> {
      const refund = await client.post(
        `/payments/${providerReference}/refund`,
        { body: { amount } },
      )
      // Moyasar reports both partial and full refunds with status "refunded".
      // Core decides PARTIALLY_REFUNDED vs REFUNDED from Σrefund vs Σcapture;
      // the adapter just reports the refund event.
      return mapMoyasarOutcome(refund) === "refunded"
        ? { outcome: "refunded", providerReference: refund.id, amount: refund.refunded ?? amount }
        : { outcome: "failed",   reason: reason ?? refund.source?.message ?? "refund failed" }
    },

    async cancel(providerReference): Promise<CancelResult> {
      const payment = await client.post(`/payments/${providerReference}/void`)
      return mapMoyasarOutcome(payment) === "voided"
        ? { outcome: "voided", providerReference: payment.id }
        : { outcome: "failed", reason: payment.source?.message ?? "void failed" }
    },

    async verifyWebhook(payload, signature) {
      return verifyMoyasarWebhook(payload, signature, options.webhookSecret)
    },
  }
}
```

Notes:

- `ctx.amount` arrives as integer minor units. Do not multiply, do not parse to float, do not call `toFixed`. The money rule in [`CLAUDE.md`](../../CLAUDE.md) is hard.
- `ctx.idempotencyKey` is set by core for every adapter call. Forward it to Moyasar so retries are safe.
- `metadata` carries `order_id`, `merchant_id`, `branch_id`. Tenancy ids are propagated even when tenancy is disabled — they will simply be `null` in that case. This is the "carry merchant and branch from the start" rule.
- Adapter errors translate into `CommerceProviderError` so the [error contract](../architecture/15-errors.md) is preserved across providers.

## Outcome mapping

Maps a Moyasar payload to the neutral outcome union the adapter returns. The adapter does **not** distinguish `PARTIALLY_REFUNDED` from `REFUNDED` or `INITIATED` from `REQUIRES_ACTION` past this point — those distinctions are derived state computed by core from the `paymentTransaction` ledger and `order.status`. See [50-adapter-system.md](../architecture/50-adapter-system.md#payment-status-is-derived-not-stored).

```ts
// src/map-outcome.ts
import { CommerceProviderError } from "commerce-kit/errors"
import type { MoyasarPaymentPayload } from "./types"

export type MoyasarOutcome =
  | "authorized"
  | "captured"
  | "refunded"
  | "voided"
  | "requires_action"
  | "failed"

export function mapMoyasarOutcome(p: MoyasarPaymentPayload): MoyasarOutcome {
  switch (p.status) {
    case "initiated":
      // "initiated" with a transaction_url means the customer must complete
      // 3-D Secure or a hosted page. No money has moved.
      return p.source?.transaction_url ? "requires_action" : "failed"
    case "paid":        return "captured"
    case "authorized":  return "authorized"
    case "captured":    return "captured"
    case "refunded":    return "refunded"
    case "voided":      return "voided"
    case "failed":
    case "expired":     return "failed"
    default:
      throw new CommerceProviderError(
        `Unknown Moyasar payment status: ${p.status}`,
        { providerId: "moyasar", providerStatus: p.status },
      )
  }
}
```

A defaulted-to-`"failed"` mapping is tempting but wrong: a new Moyasar status would silently degrade orders. Throw and let core record a `provider_error` event; the surface is then visible in logs and dashboards.

`"initiated"` without a `transaction_url` collapses to `"failed"` rather than a dedicated state — by the time `authorize()` returns, either the customer has an action to take (`requires_action`) or the attempt is dead. `INITIATED` as a derived view is only meaningful while `order.status = placed` and no transactions exist, which is the state core represents implicitly without any adapter signal.

## Webhook verification

```ts
// src/webhook.ts
import { createHmac, timingSafeEqual } from "node:crypto"
import type { WebhookEvent } from "commerce-kit"
import { CommerceWebhookError } from "commerce-kit/errors"
import { mapMoyasarOutcome } from "./map-outcome"

export async function verifyMoyasarWebhook(
  payload: unknown,
  signature: string,
  secret: string,
): Promise<WebhookEvent> {
  const raw = typeof payload === "string" ? payload : JSON.stringify(payload)
  const expected = createHmac("sha256", secret).update(raw).digest("hex")

  const given = Buffer.from(signature ?? "", "hex")
  const want = Buffer.from(expected, "hex")
  if (given.length !== want.length || !timingSafeEqual(given, want)) {
    throw new CommerceWebhookError("Invalid Moyasar webhook signature")
  }

  const event = JSON.parse(raw) as MoyasarWebhookPayload

  return {
    eventId: event.id,                       // for idempotency at the core layer
    providerReference: event.data.id,        // transaction reference
    orderId: event.data.metadata?.order_id,  // round-tripped through Moyasar.metadata
    outcome: mapMoyasarOutcome(event.data),  // 'authorized' | 'captured' | 'refunded' | 'voided' | 'failed'
    amount: event.data.amount,
    currency: event.data.currency,
    raw: event,
  }
}
```

Notes:

- The webhook layer separates the **inbound event id** (`event.id`) used for idempotency from the **transaction reference** (`event.data.id`) used by `capture` / `refund` / `cancel`. Core enforces this distinction; see [50-adapter-system.md](../architecture/50-adapter-system.md).
- The webhook resolves back to a Commerce Kit order via `event.data.metadata.order_id` — the same metadata field set on the `POST /payments` call. There is no `order.paymentReference` to look up because the order doesn't persist one.
- Core consumes `outcome`: `captured` appends a `CAPTURE` row, `refunded` a `REFUND` row, `voided` a `VOID` row, `failed` transitions the order to `failed`. The webhook adapter does not insert or transition anything itself.
- `requires_action` is intentionally not emitted from webhooks. By the time a webhook fires, the customer-action stage is over.

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
      secretKey: process.env.MOYASAR_SECRET_KEY!,        // sk_test_… for sim, sk_live_… for prod
      webhookSecret: process.env.MOYASAR_WEBHOOK_SECRET!,
      sources: ["creditcard", "applepay"],
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

Webhook mount (framework adapter rules in [80-framework-adapters.md](../architecture/80-framework-adapters.md)):

```text
POST /webhooks/payment/moyasar
```

Multiple Moyasar accounts (e.g. one per merchant) register as separate instances with distinct ids:

```ts
payments: [
  moyasar({ id: "moyasar-sa", secretKey: env.MOYASAR_SA_KEY, webhookSecret: env.MOYASAR_SA_HOOK }),
  moyasar({ id: "moyasar-ae", secretKey: env.MOYASAR_AE_KEY, webhookSecret: env.MOYASAR_AE_HOOK }),
]
```

The compile-time duplicate-id check ([50-adapter-system.md](../architecture/50-adapter-system.md)) catches collisions at the `createCommerce()` call site.

## Test plan

These are the cases the test suite must cover before shipping. Write them before the adapter code per [`CLAUDE.md`](../../CLAUDE.md).

Authorize:
- token-based credit card returns `{ outcome: 'requires_action', paymentUrl }` when 3-D Secure is required, and no `paymentTransaction` row is inserted.
- token-based credit card returns `{ outcome: 'captured' }` when `autoCapture: true`, and core appends one `CAPTURE` row.
- declined card returns `{ outcome: 'failed', reason }`, no transaction row, and `order.status` transitions to `failed`.
- unsupported currency rejects synchronously before any network call.
- idempotency key is forwarded; identical replay returns the original payment.

Capture / void / refund:
- partial capture amounts are forwarded as integer halalas; one `CAPTURE` row is appended.
- a `VOID` against a transaction whose latest type is `AUTHORIZE` returns `{ outcome: 'voided' }` and appends a `VOID` row.
- partial refund returns `{ outcome: 'refunded', amount }` and appends one `REFUND` row; existing rows untouched. Core derives `PARTIALLY_REFUNDED` from `Σrefund < Σcapture`.
- second refund that closes the gap pushes the derived state to `REFUNDED` without any adapter awareness.
- provider 4xx error becomes `CommerceProviderError` with the Moyasar code preserved.

Webhook:
- valid signature parses into a neutral `WebhookEvent`.
- tampered body or wrong secret rejects with `CommerceWebhookError`.
- replay of the same `event.id` is a no-op at the core layer (covered by the core webhook idempotency test, but verified end-to-end here).
- unknown event types map to `"payment.other"` rather than throwing.

Tenancy:
- `merchant_id` and `branch_id` appear in Moyasar `metadata` when set.
- single-store config still produces `metadata` with explicit `null`s — no schema drift between tenancy modes.

Money:
- `1234` halalas stays `1234` end-to-end. No `* 100`, no `/ 100`, no `Number` parsing.
- mixed-currency cart is rejected before reaching `authorize` (core invariant; adapter must not paper over it).
