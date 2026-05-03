# Shopify API Architecture Research

## Purpose

This document captures developer experience and architecture lessons from Shopify for Commerce Kit. Shopify is a hosted platform rather than an embeddable TypeScript commerce engine, but it is a strong reference for API surface separation, GraphQL design, app extension architecture, cart/checkout boundaries, versioning, scopes, webhooks, pagination, and rate limits.

## Sources Reviewed

- Shopify Storefront API reference.
- Shopify GraphQL Admin API reference.
- Shopify app authentication and authorization docs.
- Shopify API access scopes docs.
- Shopify API limits docs.
- Shopify GraphQL pagination docs.
- Shopify API versioning docs.
- Shopify webhook docs.
- Shopify app extension docs.
- Shopify cart Storefront API docs.

## API Surface Separation

Shopify separates APIs by audience and capability:

- Storefront API for buyer-facing product, cart, and checkout experiences.
- Admin API for store management and operational workflows.
- Customer Account API for authenticated customer data.
- Webhooks for asynchronous event delivery.
- App extensions for embedded platform behavior.

### Commerce Kit Implication

Commerce Kit should not expose one flat client. It should expose surface-specific clients and permissions.

```ts
client.store.products.list(...)
client.store.cart.create(...)
client.admin.orders.search(...)
client.admin.products.update(...)
client.webhooks.verify(...)
```

This makes data sensitivity obvious and reduces accidental storefront exposure of admin operations.

## GraphQL Design

Shopify's Storefront and Admin APIs are GraphQL-first. This gives typed queries, fragments, introspection, schema evolution, and precise data selection. Mutation failures often return mutation-level `userErrors` in addition to top-level GraphQL errors.

### Commerce Kit Implication

Commerce Kit does not need to be GraphQL-only, but it should copy the error discipline.

```ts
type MutationResult<T> =
  | { ok: true; data: T }
  | { ok: false; errors: CommerceUserError[] }
```

If Commerce Kit exposes GraphQL, clients must normalize GraphQL `errors` and domain `userErrors` because HTTP `200` does not guarantee success.

## Versioning

Shopify uses date-based API versions with regular release cadence and deprecation windows. API version, app version, and extension version are separate concerns.

### Commerce Kit Implication

Commerce Kit should separate package versions from API contract versions and webhook payload versions.

Recommended persisted metadata:

```ts
apiVersion: "2026-04"
webhookVersion: "2026-04"
pluginVersion: "1.2.0"
```

Even if Commerce Kit starts without public hosted APIs, versioning discipline matters for generated clients and persisted webhook events.

## Cart And Checkout

Shopify treats cart as a secure buyer-facing primitive. Cart APIs cover creating carts, updating lines, setting buyer identity, delivery options, and obtaining a checkout URL. Cart IDs can include secret-bearing key material and must be treated carefully.

### Commerce Kit Implication

Commerce Kit should treat carts as secure session-like objects, not plain public IDs.

```ts
await client.store.cart.create({ lines })
await client.store.cart.setBuyerIdentity(cartId, buyerIdentity)
await client.store.cart.getCheckoutUrl(cartId)
```

Cart IDs should be opaque. If Commerce Kit supports public cart IDs, they should be scoped, signed, or otherwise protected.

## App Extensions

Shopify app extensions let apps extend platform UI and behavior through deployment-aware configuration. Extension versions are bundled with app deployment state.

### Commerce Kit Implication

Commerce Kit plugins should have declarative manifests that describe capabilities, surfaces, schema changes, routes, permissions, and UI/admin contributions.

```ts
defineCommercePlugin({
  id: "reviews",
  surfaces: ["store", "admin"],
  permissions: ["ProductRead", "ReviewWrite"],
  routes: { admin: [...], store: [...] },
})
```

## Webhooks

Shopify webhooks are HMAC-signed, versioned, retried, and not guaranteed to arrive exactly once or in order.

### Commerce Kit Implication

Commerce Kit webhook handling should be idempotent by default:

- Verify signature before parsing trusted payloads.
- Persist event IDs for dedupe.
- Decode by payload version.
- Support retry and replay.
- Avoid order-sensitive assumptions.

## Pagination And Limits

Shopify GraphQL uses cursor connections with `first/after`, `last/before`, and `pageInfo`. Admin GraphQL also uses cost-based throttling and exposes cost information in response extensions.

### Commerce Kit Implication

Commerce Kit should standardize cursor pagination and expose iteration helpers.

```ts
for await (const order of client.admin.orders.iterate({ status: "paid" })) {
  await syncOrder(order)
}
```

Commerce Kit should also expose rate-limit and query-cost metadata when adapters can provide it.

## Auth And Scopes

Shopify uses explicit access scopes across APIs and app surfaces. Scope approval and protected customer data create friction but protect sensitive operations.

### Commerce Kit Implication

Commerce Kit should require explicit admin capability declarations and surface them in generated handlers and docs.

```ts
permissions.required("OrderRefund")
```

## Strengths

- Clear API surface separation.
- Strong GraphQL typing and schema introspection.
- Mature versioning and deprecation model.
- Good cursor pagination conventions.
- App extensions are deployment-aware.
- Webhooks are designed for duplicate/out-of-order delivery.
- Scopes make capability boundaries visible.

## Weaknesses

- Many auth modes increase cognitive load.
- GraphQL errors can be missed if clients only inspect HTTP status.
- Cost limits and input limits create hidden constraints.
- Cart IDs and customer data require security care.
- App/API/extension versioning can be hard to reason about.

## Commerce Kit Priority Takeaways

- Split Store, Admin, Customer, and Webhook surfaces.
- Normalize GraphQL/domain errors if GraphQL is used.
- Treat carts as secure primitives.
- Make webhooks idempotent and version-aware.
- Use cursor pagination and expose iteration helpers.
- Separate API versions from package/plugin versions.
