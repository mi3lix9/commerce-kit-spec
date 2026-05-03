# Saleor API Architecture Research

## Purpose

This document captures API and architecture lessons from Saleor for Commerce Kit. Saleor is a GraphQL-first headless commerce platform with strong patterns around checkout, channels, apps, webhooks, permissions, metadata, payments, shipping, and code generation.

## Sources Reviewed

- Saleor API overview: `https://docs.saleor.io/api-usage/overview`
- Saleor API reference: `https://docs.saleor.io/api-reference/`
- Saleor error handling: `https://docs.saleor.io/api-usage/error-handling`
- Saleor metadata: `https://docs.saleor.io/api-usage/metadata`
- Saleor checkout overview: `https://docs.saleor.io/developer/checkout/overview`
- Saleor channels: `https://docs.saleor.io/developer/channels/overview`
- Saleor permissions: `https://docs.saleor.io/developer/permissions`
- Saleor payments and transactions docs.
- Saleor apps and webhooks docs.

## GraphQL-First Contract

Saleor is schema-centered. The API is GraphQL, and developer experience relies heavily on schema introspection, generated types, typed operations, and consistent mutation payloads.

### Commerce Kit Implication

Commerce Kit should be schema-first whether it exposes REST, RPC, or GraphQL. Generated clients should come from the configured Commerce Kit instance and plugin set.

```ts
const client = createCommerceClient<typeof commerce>()
```

If Commerce Kit provides GraphQL, it should document codegen as the default path rather than optional advanced setup.

## Checkout Model

Saleor treats checkout as a primary domain object. It contains cart-like state, discounts, addresses, shipping, payment state, and completion behavior.

### Commerce Kit Implication

Commerce Kit should decide whether `cart` and `checkout` are separate or whether checkout is the evolved cart aggregate. The safest model is:

- `cart` for browsing and line item changes.
- `checkout` for buyer identity, addresses, shipping selection, payment, and finalization.
- `order` only after durable placement.

```ts
await client.store.carts.addItem(cartId, line)
await client.store.checkouts.start({ cartId })
await client.store.checkouts.complete(checkoutId)
```

## Channels

Saleor channels affect pricing, stock, shipping, discounts, availability, and access. Channels are central to headless commerce in multi-region and multi-brand deployments.

### Commerce Kit Implication

Commerce Kit should make channel context explicit in storefront calls.

```ts
await client.store.products.list({ channel: "us" })
```

Channel should participate in cache keys, pricing, shipping, tax, payments, and permissions.

## Apps And Webhooks

Saleor's extension model favors apps and webhooks over modifying core. Apps have permissions and can receive asynchronous and synchronous webhooks. Synchronous webhooks can participate in request-critical decisions such as shipping or transaction processing.

### Commerce Kit Implication

Commerce Kit should distinguish sync extension points from async side effects.

```ts
hooks: {
  "shipping.rates.calculate": { mode: "sync", handler },
  "order.paid": { mode: "async", handler },
}
```

Synchronous hooks should have strict timeouts and failure semantics. Async hooks should use queues, retries, and replay.

## Payments And Transactions

Saleor's transaction model supports sync and async payment flows, retries, refunds, payment method details, and events. This is more flexible than treating payment as a single status string.

### Commerce Kit Implication

Commerce Kit should model payment attempts and transaction events separately from the order.

```ts
type Payment = { id: string; orderId: string; state: PaymentState }
type PaymentEvent = { paymentId: string; type: string; amount?: Money }
```

This supports delayed confirmation, failed capture, partial refunds, disputes, and provider reconciliation.

## Permissions

Saleor has explicit staff/app permissions and channel restrictions. Permission groups are additive, which can broaden access unexpectedly.

### Commerce Kit Implication

Commerce Kit should define permission presets and avoid implicit broadening.

```ts
roles: {
  support: ["OrderRead", "CustomerRead"],
  refunds: ["OrderRead", "RefundCreate"],
}
```

Admin APIs should include route-level permission metadata.

## Metadata

Saleor metadata is useful for extension data, external IDs, and display hints. It should not become a dumping ground for sensitive or core state.

### Commerce Kit Implication

Commerce Kit should support namespaced metadata but clearly separate it from typed custom fields and core columns.

```ts
metadata: {
  "loyalty.pointsAwarded": "100",
}
```

## Error Model

Saleor GraphQL mutations return structured errors that include fields and codes. This makes form handling and domain error handling more predictable.

### Commerce Kit Implication

Commerce Kit mutations should return stable error codes and field paths.

```ts
{ code: "INSUFFICIENT_STOCK", field: "lines.0.quantity" }
```

## Strengths

- Strong GraphQL and codegen developer experience.
- Checkout is a first-class aggregate.
- Channel model is deeply integrated.
- Apps/webhooks keep core clean.
- Permission model is explicit.
- Metadata and extensions support integration needs.
- Payment transaction model handles async reality.

## Weaknesses

- Channels, apps, webhooks, and permissions create complexity.
- Synchronous webhooks can add latency and availability risks.
- Checkout/order/payment semantics can be subtle.
- Metadata can be overused.
- Breaking changes and deprecations require upgrade discipline.

## Commerce Kit Priority Takeaways

- Use schema/codegen as a first-class DX path.
- Model checkout as its own domain aggregate.
- Make channel context explicit and cache-aware.
- Prefer apps/webhooks/plugins over core edits.
- Separate synchronous extension points from async events.
- Model payment transaction events, not just payment status.
- Use structured domain errors with codes and fields.
