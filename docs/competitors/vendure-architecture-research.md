# Vendure Architecture Research

## Purpose

This document captures architecture lessons from Vendure for Commerce Kit. Vendure is one of the strongest references for a TypeScript-first, extensible commerce backend because it combines a plugin-first architecture, GraphQL API split, channels, custom fields, workers, events, migrations, and payment/shipping strategy objects.

## Sources Reviewed

- Vendure developer overview: `https://docs.vendure.io/guides/developer-guide/overview/`
- Vendure plugins: `https://docs.vendure.io/guides/developer-guide/plugins/`
- Vendure channels: `https://docs.vendure.io/guides/core-concepts/channels/`
- Vendure custom fields: `https://docs.vendure.io/guides/developer-guide/custom-fields/`
- Vendure worker and job queue: `https://docs.vendure.io/guides/developer-guide/worker-job-queue/`
- Vendure payment: `https://docs.vendure.io/guides/core-concepts/payment/`
- Vendure shipping and fulfillment: `https://docs.vendure.io/guides/core-concepts/shipping/`
- Vendure auth and permissions docs.
- Vendure migrations docs.

## Positioning

Vendure is a TypeScript commerce framework built around NestJS, GraphQL, TypeORM, and a plugin model. It is closer to Commerce Kit's intended direction than most platform APIs because it treats commerce as an extensible server application rather than only a hosted API.

The best Vendure lesson is not to copy NestJS, but to copy the discipline: commerce capabilities are modules with entities, services, API extensions, events, jobs, permissions, and migrations.

## Plugin System

Vendure plugins are effectively a superset of NestJS modules. A plugin can declare imports, providers, entities, configuration changes, Shop API extensions, Admin API extensions, and dashboard extensions.

Strong patterns:

- Plugins compose into the application at startup.
- Plugins can add database entities and GraphQL schema extensions.
- Plugins can contribute services and event handlers.
- Plugins can add custom permissions.
- Plugins are the normal way to extend core behavior.

### Commerce Kit Implication

Commerce Kit should treat plugins as first-class modules rather than loose hook collections.

```ts
const loyaltyPlugin = defineCommercePlugin({
  id: "loyalty",
  schema: { ... },
  adminApi: { ... },
  storeApi: { ... },
  permissions: ["LoyaltyRead", "LoyaltyWrite"],
  jobs: ["loyalty.points.recalculate"],
  hooks: {
    "order.paid": async ({ order }) => { ... },
  },
})
```

Plugins should be able to extend schema, API, events, jobs, and client types, but only through typed contracts.

## Shop API And Admin API Split

Vendure separates the Shop API from the Admin API. The Shop API is storefront-facing and customer-scoped. The Admin API is staff/admin-facing and permissioned.

This split prevents accidental exposure of sensitive commerce operations and lets each surface optimize for different developers.

### Commerce Kit Implication

Commerce Kit should formalize two generated clients:

```ts
client.store.products.list(...)
client.store.cart.addItem(...)
client.store.checkout.complete(...)

client.admin.products.create(...)
client.admin.orders.search(...)
client.admin.refunds.create(...)
```

Storefront APIs should be safe by default. Admin APIs should require host-app auth and explicit permissions.

## Channels

Vendure channels model multi-store, multi-region, multi-vendor, and marketplace scenarios. Channel-aware entities can vary by channel for pricing, currency, stock, roles, and visibility.

Strong patterns:

- Channel is a real commerce dimension, not just metadata.
- Products, prices, shipping methods, payment methods, and roles can be channel-aware.
- Default channels exist for simpler installs.
- Channel-aware behavior is part of core APIs.

### Commerce Kit Implication

Commerce Kit should decide early whether `channel` is core. If Commerce Kit targets serious ecommerce use cases, channel awareness should be first-class because retrofitting it later affects products, prices, inventory, orders, customers, shipping, payments, tax, and permissions.

Recommended shape:

```ts
const commerce = createCommerce({
  channels: {
    default: "us-web",
    resolve: async (request) => "us-web",
  },
})
```

## Custom Fields

Vendure custom fields allow extensions to add typed fields to core entities without forking the core schema. Custom fields integrate with GraphQL and the admin dashboard, but schema changes still require migrations.

### Commerce Kit Implication

Commerce Kit should support typed extension fields for common customization needs.

```ts
defineCommercePlugin({
  id: "product-badges",
  fields: {
    product: {
      badgeText: { type: "string", nullable: true },
    },
  },
})
```

Custom fields should be namespaced, migration-aware, and reflected in generated client types.

## Worker And Job Queue

Vendure separates the server process from worker processes. Long-running jobs run outside request/response paths. This fits commerce workloads such as emails, search indexing, imports, webhook dispatch, payment reconciliation, and fulfillment sync.

Pitfalls in Vendure include dual-runtime lifecycle complexity, database load from simple polling queues, and large serialized request contexts in jobs.

### Commerce Kit Implication

Commerce Kit should expose background jobs as a core primitive but avoid making every app adopt a heavyweight worker immediately.

Recommended model:

```ts
commerce.jobs.enqueue("order.email.send", { orderId })
commerce.jobs.process("order.email.send", async ({ orderId }) => { ... })
```

Adapters should support in-memory development queues and production queue backends.

## Payment And Shipping Strategies

Vendure models payments and shipping through handlers, calculators, checkers, and strategies. Integrations plug into commerce workflows without owning the whole order lifecycle.

### Commerce Kit Implication

Commerce Kit should model payments and shipping as strategy objects with capabilities.

```ts
type PaymentAdapter = {
  id: string
  capabilities: PaymentCapabilities
  authorize(input: AuthorizeInput): Promise<PaymentResult>
  capture(input: CaptureInput): Promise<PaymentResult>
  refund(input: RefundInput): Promise<RefundResult>
}
```

Provider adapters should not mutate order state directly. They should return normalized results, and core should transition orders.

## Permissions

Vendure supports built-in and custom permissions. Plugins can define permission names and use them in Admin API extensions.

### Commerce Kit Implication

Commerce Kit should define permission primitives before admin APIs become large.

```ts
permissions: [
  "ProductRead",
  "ProductWrite",
  "OrderRead",
  "OrderRefund",
  "Plugin:loyalty:Write",
]
```

Permissions should be typed and surfaced in route guards/framework adapters.

## Strengths

- Strong TypeScript-first architecture reference.
- Plugin model covers entities, APIs, services, jobs, and config.
- Shop/Admin API split is a strong boundary.
- Channels are deeply integrated.
- Custom fields solve common extension use cases.
- Worker process model fits commerce operations.
- Payment and shipping strategies avoid hardcoded providers.

## Weaknesses

- Conceptual load is high.
- Server/worker dual runtime adds operational complexity.
- Custom fields and plugin entities require migration discipline.
- GraphQL extensions can duplicate work across Shop and Admin APIs.
- Some strategy integrations are lower-level than app developers expect.

## Commerce Kit Priority Takeaways

- Make plugins modular and capability-rich, not just callback bags.
- Split Store and Admin APIs from the start.
- Treat channels as a first-class commerce dimension if multi-store support matters.
- Add typed custom fields for extension data.
- Make background jobs and event handling core infrastructure.
- Keep payment and shipping adapters strategy-based.
- Define typed permissions early.
