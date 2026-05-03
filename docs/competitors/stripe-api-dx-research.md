# Stripe API DX Research

## Purpose

This document captures developer experience lessons from Stripe for Commerce Kit. Stripe is not an ecommerce engine, but it is a leading reference for financial API rigor: idempotency, webhooks, versioning, structured errors, request IDs, retries, test tooling, pagination, metadata, and payment lifecycle design.

## Sources Reviewed

- Stripe Node SDK docs and README.
- Stripe API docs for PaymentIntents, Checkout Sessions, Refunds, Disputes, Metadata, Errors, Rate Limits, Request IDs, Webhooks, and Testing.
- Stripe CLI docs.

## SDK Ergonomics

Stripe's Node SDK exposes resource namespaces with async methods and typed parameters. It supports per-request options such as idempotency key, timeout, retry settings, API version, connected account, and custom transport settings.

### Commerce Kit Implication

Commerce Kit adapters should centralize provider SDK setup. App code should not instantiate Stripe/Square/Adyen clients directly for core commerce flows.

```ts
const commerce = createCommerce({
  payments: stripeAdapter({
    apiVersion: "2025-02-24.acacia",
    secretKey: env.STRIPE_SECRET_KEY,
  }),
})
```

## Idempotency

Stripe's idempotency model is one of its strongest DX features. Writes can include idempotency keys so network retries do not duplicate charges, refunds, or other financial mutations.

### Commerce Kit Implication

Commerce Kit should make idempotency mandatory for financial and order-completion writes.

High-priority idempotent operations:

- checkout completion
- order creation
- payment authorization
- payment capture
- payment cancellation
- refund creation
- webhook processing

Commerce Kit should generate deterministic keys from business IDs when callers do not provide one.

## Payment Lifecycle

Stripe's PaymentIntent lifecycle models asynchronous payment reality: requires payment method, requires confirmation, requires action, processing, succeeded, canceled, and more. Checkout Sessions provide a higher-level hosted checkout abstraction.

### Commerce Kit Implication

Commerce Kit should expose simple checkout APIs while internally preserving a payment state machine.

```ts
await commerce.checkout.sessions.create({ cartId, mode: "hosted" })
await commerce.payments.authorize({ orderId, paymentMethod })
await commerce.payments.capture(paymentId)
```

Provider-specific statuses should map into Commerce Kit states instead of leaking into order logic.

## Webhooks

Stripe makes webhooks first-class. Signature verification requires raw request bodies. The CLI supports local forwarding, event triggering, and replay-like workflows.

### Commerce Kit Implication

Commerce Kit framework adapters should own raw-body-safe webhook handlers.

```ts
export const POST = commerce.webhooks.handler("stripe", {
  onEvent: async (event) => { ... },
})
```

Webhook processing should verify signatures, dedupe event IDs, persist payloads, and reconcile payment state.

## Errors And Request IDs

Stripe has structured errors with types, codes, decline codes, request IDs, and HTTP status. Request IDs are critical for support and debugging.

### Commerce Kit Implication

Commerce Kit should normalize provider errors into stable domain codes while preserving provider details.

```ts
class CommercePaymentError extends CommerceError {
  code: "PAYMENT_DECLINED" | "PAYMENT_REQUIRES_ACTION" | "PROVIDER_UNAVAILABLE"
  providerRequestId?: string
  providerCode?: string
}
```

All payment logs should include provider request IDs where available.

## API Versioning

Stripe pins API versions and allows explicit version upgrades. Webhook endpoints also have versioned payload behavior.

### Commerce Kit Implication

Payment adapters should declare and persist provider API versions used for important objects and events.

```ts
payment.providerApiVersion = "2025-02-24.acacia"
webhook.payloadVersion = "2025-02-24.acacia"
```

This helps with long-lived orders and replayed events.

## Pagination

Stripe list APIs are cursor-based and SDKs provide auto-pagination helpers. Auto-pagination is ergonomic but can hide expensive or memory-heavy work.

### Commerce Kit Implication

Commerce Kit should offer both page-based responses and explicit iterators with safety limits.

```ts
const page = await client.admin.orders.list({ limit: 50 })

for await (const order of client.admin.orders.iterate({ limit: 100 })) {
  await sync(order)
}
```

## Expansions And Includes

Stripe lets callers expand related objects. This is powerful, but in TypeScript it often creates `string | object` unions that require narrowing.

### Commerce Kit Implication

Commerce Kit should avoid leaking provider expansion typing into app code. Domain APIs should return normalized Commerce Kit objects.

If Commerce Kit supports relation loading, it should be typed explicitly:

```ts
await client.admin.orders.get(orderId, {
  include: ["items", "payments", "customer"],
})
```

## Metadata

Stripe metadata is useful for external IDs and workflow references, but it is string-only, size-limited, and not for sensitive data.

### Commerce Kit Implication

Commerce Kit should reserve provider metadata keys and enforce safe values.

Recommended keys:

- `commercekit_order_id`
- `commercekit_payment_id`
- `commercekit_tenant_id`
- `commercekit_workflow_id`

## CLI And Test Tooling

Stripe CLI is a major DX advantage. Developers can listen for webhooks, forward events locally, trigger known event types, and inspect API behavior.

### Commerce Kit Implication

Commerce Kit CLI should eventually support local webhook forwarding, fixture generation, event replay, and provider test-mode verification.

## Strengths

- Strong financial API ergonomics.
- Idempotency is well documented and practical.
- Webhook verification and CLI tooling are first-class.
- Structured errors and request IDs are excellent for support.
- API version pinning supports long-lived integrations.
- Metadata helps connect provider objects to app objects.

## Weaknesses

- Payment lifecycle complexity is unavoidable.
- Webhook raw body handling is a common footgun.
- Expansion typing can be noisy.
- Test mode does not cover every production edge case.
- Auto-pagination can hide cost.

## Commerce Kit Priority Takeaways

- Make idempotency core infrastructure.
- Treat webhooks as the source of truth for payment reconciliation.
- Normalize provider states into Commerce Kit payment states.
- Preserve request IDs and provider codes in errors/logs.
- Provide CLI/dev tooling for webhook testing.
- Keep provider SDK complexity behind adapters.
