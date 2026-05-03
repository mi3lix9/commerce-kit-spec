# commercetools Architecture Research

## Purpose

This document captures architecture lessons from commercetools Composable Commerce for Commerce Kit. commercetools is an enterprise composable commerce platform with strong patterns around versioned resources, optimistic concurrency, command-style updates, states, stores, channels, custom fields, product projections, subscriptions, extensions, import APIs, and SDK middleware.

## Sources Reviewed

- commercetools HTTP API resource management docs.
- commercetools authorization and scopes docs.
- commercetools errors docs.
- commercetools query predicates docs.
- commercetools product projections docs.
- commercetools channels and stores docs.
- commercetools types/custom fields docs.
- commercetools states docs.
- commercetools API extensions and subscriptions docs.
- commercetools Import API docs.
- commercetools TypeScript SDK best practices.

## Versioned Resources And Update Actions

commercetools writes use optimistic concurrency. Resources have `id`, `key`, and `version`. Updates include the expected `version` plus an array of update actions. A stale version produces a conflict, which callers must handle.

```ts
POST /orders/{id}
{
  "version": 7,
  "actions": [
    { "action": "changeOrderState", "orderState": "Complete" }
  ]
}
```

### Commerce Kit Implication

Commerce Kit should use versioned writes for mutable commerce aggregates such as carts, orders, payments, inventory, and customer profiles.

```ts
await commerce.orders.update(orderId, {
  version,
  actions: [{ type: "markPaid", paymentId }],
})
```

This is more durable than generic patch objects because commands encode intent.

## Project, Store, And Channel Boundaries

commercetools uses Projects as broad isolation boundaries. Stores and Channels model commerce context such as storefronts, regions, distribution, supply, and inventory concerns.

### Commerce Kit Implication

Commerce Kit should not overload one `tenantId` for every context. It should distinguish:

- tenant/project: application or business boundary
- store/channel: commercial context
- inventory channel: supply context
- locale/currency: presentation and pricing context

## Product Read/Write Separation

commercetools separates write models from read-optimized product projections/search. Products are the source of truth, while projections/search are optimized for storefront reads.

### Commerce Kit Implication

Commerce Kit should consider read models early.

```ts
admin.products.update(...)
store.products.search(...)
```

Storefront search should not necessarily query the same shape used for product administration.

## Types And Custom Fields

commercetools Types and Custom Fields allow schema extension across resources. This is powerful but can become hard to govern if every integration adds unstructured fields.

### Commerce Kit Implication

Commerce Kit should support typed extension fields with ownership and namespace metadata.

```ts
fields: {
  order: {
    "erp.externalId": { type: "string", owner: "erp" },
  },
}
```

Custom fields should not replace core domain modeling.

## States

commercetools States model workflow state machines for entities such as orders, payments, products, and reviews.

### Commerce Kit Implication

Commerce Kit should centralize state machines instead of scattering status strings.

Important state machines:

- cart lifecycle
- order lifecycle
- payment lifecycle
- refund lifecycle
- fulfillment lifecycle
- inventory reservation lifecycle

## Extensions And Subscriptions

commercetools API Extensions are synchronous and can validate or enrich requests, but they block the originating API call. Subscriptions are asynchronous and better for downstream side effects.

### Commerce Kit Implication

Commerce Kit should have two extension modes:

- synchronous hooks for validation, pricing, shipping rates, and request-critical decisions
- asynchronous events for indexing, email, ERP sync, webhook dispatch, and analytics

Synchronous hooks need strict timeout, retry, and fallback semantics.

## Import And Sync APIs

commercetools has async import APIs for bulk ingestion and project sync tooling for environment promotion. Imports are eventually consistent and should not be used for immediate read-after-write flows.

### Commerce Kit Implication

Commerce Kit should separate transactional writes from bulk import/sync workflows.

```ts
await commerce.import.products.enqueue(batch)
await commerce.import.status(jobId)
```

Bulk import should expose status, partial failures, and replayable batches.

## Query Predicates And SDKs

commercetools supports query predicates and SDK middleware. This is powerful but easy to misuse if applications build arbitrary query strings.

### Commerce Kit Implication

Commerce Kit should provide typed filter builders for admin/search APIs.

```ts
client.admin.orders.list({
  where: orderWhere.status.in(["paid", "fulfilled"]),
})
```

This keeps filtering expressive without sacrificing safety.

## Pagination And Limits

commercetools offset pagination has caps and totals may not be strongly consistent. Search and sync APIs should be used for large datasets.

### Commerce Kit Implication

Commerce Kit should document pagination limits and provide cursor/iterator helpers for large data sync.

## Auth And Scopes

commercetools scopes are granular and can be resource/store-specific. Broad scopes such as project-wide management should be avoided in production.

### Commerce Kit Implication

Commerce Kit should represent adapter credentials and permissions explicitly.

```ts
adapter.capabilities.requiredScopes
```

The CLI could eventually validate missing provider scopes.

## Strengths

- Strong optimistic concurrency model.
- Command-style update actions encode intent.
- Store/channel/state/type concepts are mature.
- Read/write separation supports scale.
- Subscriptions and imports separate async workloads.
- Granular scopes support least privilege.

## Weaknesses

- High conceptual complexity.
- Conflict retry/merge logic is mandatory.
- Offset pagination caps require planning.
- Synchronous extensions can hurt availability.
- Custom field sprawl is a governance risk.
- Enterprise API shape can feel verbose for small apps.

## Commerce Kit Priority Takeaways

- Use versioned writes for carts, orders, payments, and inventory.
- Prefer command-style update actions over generic patches for core aggregates.
- Separate storefront read models from admin write models.
- Model states explicitly.
- Treat sync hooks and async events differently.
- Build import/sync APIs separately from transactional APIs.
- Keep scopes and credentials least-privilege.
