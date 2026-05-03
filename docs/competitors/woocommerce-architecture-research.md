# WooCommerce Architecture Research

## Purpose

This document captures architecture lessons from WooCommerce for Commerce Kit. WooCommerce is not a TypeScript SDK reference, but it is one of the strongest architecture references for a mature open-source commerce ecosystem: plugin lifecycle, event-driven extension points, payment and shipping provider contracts, order state transitions, public storefront APIs, admin APIs, data-store abstraction, migration compatibility, and marketplace-level extension rules.

The main takeaway is not to copy WooCommerce's WordPress/PHP architecture. The useful lesson is how a commerce core can survive long-term extensibility: stable domain contracts, explicit extension lifecycle, clear compatibility boundaries, public/private API separation, and migration paths for persisted commerce data.

## Sources Reviewed

- WooCommerce merchant documentation: `https://woocommerce.com/documentation/woocommerce/`
- WooCommerce developer docs: `https://developer.woocommerce.com/docs/`
- Project structure: `https://developer.woocommerce.com/docs/getting-started/project-structure/`
- Extension design: `https://developer.woocommerce.com/docs/extensions/getting-started-extensions/how-to-design-a-simple-extension/`
- Extension best practices: `https://developer.woocommerce.com/docs/extensions/best-practices-extensions/extension-development-best-practices/`
- Payment Gateway API: `https://developer.woocommerce.com/docs/features/payments/payment-gateway-api/`
- Shipping Method API: `https://developer.woocommerce.com/docs/features/shipping/shipping-method-api/`
- High Performance Order Storage: `https://developer.woocommerce.com/docs/features/high-performance-order-storage/`
- Store API: `https://developer.woocommerce.com/docs/apis/store-api/`
- REST API docs: `https://woocommerce.github.io/woocommerce-rest-api-docs/`
- Code reference: `https://woocommerce.github.io/code-reference/`

## Positioning

WooCommerce is an open-source ecommerce platform built on WordPress. It organizes merchant-facing functionality into products, taxes, shipping, payments, orders, reporting, customers, marketing, administration, store design, and mobile operations.

For developers, WooCommerce treats extensions as specialized WordPress plugins. Extension authors are expected to understand hooks, filters, plugin lifecycle, WordPress coding standards, WooCommerce data objects, and WooCommerce APIs.

### Commerce Kit Implication

Commerce Kit should position itself differently: a library-first TypeScript commerce engine, not a monolithic application. Still, WooCommerce validates the need for a clear extension ecosystem, stable domain contracts, and first-class docs for payment, shipping, storage, webhooks, storefront APIs, admin APIs, and lifecycle behavior.

## Monorepo And Package Boundaries

WooCommerce's repository is a monorepo containing plugins, packages, and tools. The core WooCommerce plugin lives under `plugins/woocommerce`. The `packages` directory contains PHP and JavaScript packages, some of which are internal. The `tools` directory contains monorepo utilities.

WooCommerce warns developers not to clone the full monorepo directly into `wp-content/plugins`; the plugin is one part of a larger development structure.

### Commerce Kit Implication

Commerce Kit should keep package boundaries explicit from the start.

Recommended package model:

```txt
packages/core
packages/client
packages/adapters-drizzle
packages/adapters-stripe
packages/adapters-square
packages/framework-next
packages/plugin-subscriptions
packages/plugin-digital-products
packages/cli
```

The docs should distinguish public packages from internal packages. Internal modules should be physically and semantically hard to import from app code.

## Extension Lifecycle

WooCommerce extensions have a main plugin file that acts as the entry point and lifecycle hub. It declares metadata, handles activation, execution, deactivation, and uninstallation, loads dependencies, initializes classes, and delays initialization until WordPress and WooCommerce are loaded.

Important extension lifecycle patterns:

- Declare extension metadata in one place.
- Prevent direct execution of extension files.
- Register activation and deactivation hooks.
- Delay initialization until dependencies are loaded.
- Check that WooCommerce is active before doing work.
- Load dependencies before registering runtime hooks.
- Keep deactivation separate from uninstallation because merchants may deactivate temporarily.

### Commerce Kit Implication

Commerce Kit plugins need a lifecycle contract, even if they are plain TypeScript objects rather than installed WordPress plugins.

Recommended plugin shape:

```ts
const plugin = defineCommercePlugin({
  id: "subscriptions",
  version: "1.0.0",
  dependsOn: ["payments"],
  schema: { ... },
  init(ctx) { ... },
  onInstall(ctx) { ... },
  onMigrate(ctx) { ... },
  onDisable(ctx) { ... },
})
```

Commerce Kit should define exactly when plugin lifecycle hooks run, which services are available at each phase, and which hooks can mutate schema, routes, client types, or runtime behavior.

## Event-Driven Extension Model

WooCommerce is deeply event-driven through WordPress actions and filters. Extensions add or modify behavior by registering callbacks for specific events. Payment gateways are registered through the `woocommerce_payment_gateways` filter. Shipping methods are registered through `woocommerce_shipping_methods`. Shipping rates can be extended with actions such as `woocommerce_{id}_shipping_add_rate`.

WooCommerce extension docs explicitly recommend an event-driven mindset because merchant and shopper behavior flows through web requests and state transitions.

### Commerce Kit Implication

Commerce Kit should expose typed domain events and hooks rather than generic stringly typed filters.

Recommended model:

```ts
const commerce = createCommerce({
  plugins: [
    defineCommercePlugin({
      id: "loyalty",
      hooks: {
        "cart.item.added": async ({ cart, item }) => { ... },
        "order.paid": async ({ order, payment }) => { ... },
      },
    }),
  ],
})
```

Hooks should be typed, scoped, and ordered. Commerce Kit should avoid arbitrary mutation hooks where structured extension points are safer.

## Public And Internal API Boundaries

WooCommerce best practices explicitly warn extension developers not to use classes in the `Automattic\WooCommerce\Internal` namespace or code marked `@internal`. Backwards compatibility is not guaranteed for internal code.

The Store API also uses a more controlled extensibility model through `ExtendSchema`, intentionally limiting how extensions can change responses to reduce breakage.

### Commerce Kit Implication

Commerce Kit needs a strict public/internal API boundary from the start.

Recommended rules:

- Public APIs are exported from package entrypoints only.
- Internal modules use explicit `internal` paths and are not documented for app use.
- Plugins extend core through declared contracts, not by importing internals.
- Schema extension should be additive and namespaced.
- Public hooks should have versioned payload contracts.

This matters more for Commerce Kit than for many libraries because commerce plugins will touch persisted data, payment lifecycle, order state, and customer-visible checkout behavior.

## Payment Gateway Architecture

WooCommerce payment gateways are class-based extensions that inherit from `WC_Payment_Gateway`. A gateway declares metadata such as ID, icon, fields, method title, method description, settings fields, and saved settings. The core payment method is `process_payment($order_id)`, which receives an order and returns a result plus redirect URL.

Gateway types include:

- Form-based gateways that redirect to an external processor.
- Iframe-based gateways that embed processor UI.
- Direct gateways that collect fields on checkout.
- Offline gateways such as cheque, bank transfer, and cash on delivery.

Important payment behavior:

- Use `payment_complete()` when payment is complete so WooCommerce handles stock and status transitions.
- Use `update_status('on-hold')` when payment requires manual verification.
- Use `failed` status when payment fails after order creation.
- Gateway callbacks use WC-API hooks, such as `woocommerce_api_wc_gateway_paypal`.
- Hooks inside gateway classes may not trigger unless the gateway is loaded, so callbacks and global hooks need careful registration.

### Commerce Kit Implication

Commerce Kit should treat payment providers as adapters with an explicit state-machine contract, not as arbitrary callbacks.

Recommended adapter capabilities:

```ts
type PaymentAdapter = {
  id: string
  capabilities: {
    hostedCheckout?: boolean
    embeddedCheckout?: boolean
    authorize?: boolean
    capture?: boolean
    refund?: boolean
    manualReview?: boolean
    offline?: boolean
  }
  createSession?(input: CreateCheckoutSessionInput): Promise<CheckoutSession>
  authorize?(input: AuthorizePaymentInput): Promise<PaymentResult>
  capture?(input: CapturePaymentInput): Promise<PaymentResult>
  cancel?(input: CancelPaymentInput): Promise<PaymentResult>
  refund?(input: RefundPaymentInput): Promise<RefundResult>
  verifyWebhook?(input: VerifyWebhookInput): Promise<WebhookEvent>
}
```

Commerce Kit should preserve WooCommerce's important order-state lesson: payment completion should trigger stock, fulfillment readiness, status transition, and event emission through core, not through provider-specific adapter code.

## Shipping Method Architecture

WooCommerce shipping methods extend `WC_Shipping_Method`. Shipping methods are registered through a filter, declare support flags, define global and per-instance settings, sanitize merchant input, and implement `calculate_shipping($package)` to add one or more shipping rates.

Important shipping patterns:

- Shipping methods are installed globally but configured per shipping zone.
- One shipping method can have multiple zone-specific instances.
- Shipping rates are calculated from a package of cart items.
- Rates include ID, label, cost, taxes, calculation mode, and package context.
- Merchant-entered costs require sanitization and locale-aware decimal handling.
- Shipping classes influence rate calculation.

### Commerce Kit Implication

Commerce Kit should separate shipping providers, shipping methods, shipping zones, shipping rates, and packages.

Recommended model:

```ts
type ShippingAdapter = {
  id: string
  listRates(input: ShippingRateInput): Promise<ShippingRate[]>
}

type ShippingRateInput = {
  cartId: string
  destination: Address
  packages: ShippingPackage[]
  currency: string
}
```

Shipping calculation should be a first-class domain service. Plugins should be able to add rates, filter rates, or mark rates unavailable without mutating cart internals directly.

## Storefront API Versus Admin API

WooCommerce has both an authenticated REST API and an unauthenticated Store API.

The Store API is for customer-facing product, cart, checkout, and current-user order functionality. It is unauthenticated, JSON-only, reflective of the current cookie-based customer session, and intentionally cannot access arbitrary customers, arbitrary orders, or store settings. Writes require nonce tokens.

The authenticated REST API is for broader store management: coupons, customers, orders, refunds, products, variations, attributes, categories, reports, tax rates, webhooks, settings, payment gateways, shipping zones, shipping methods, and system status.

### Commerce Kit Implication

Commerce Kit should explicitly split storefront and admin APIs.

```ts
client.store.products.list(...)
client.store.cart.addItem(...)
client.store.checkout.complete(...)

client.admin.products.create(...)
client.admin.orders.search(...)
client.admin.refunds.create(...)
client.admin.settings.update(...)
```

The storefront API should expose only customer-safe data. Admin APIs should require auth integration from the host app or framework adapter.

## Store API Route Architecture

WooCommerce Store API route architecture separates route mapping, schema formatting, and utility/controller access.

Key pieces:

- Route classes map requests to endpoints, handle collections, errors, pagination, and JSON responses.
- Schema classes format resources such as products, carts, cart items, and orders.
- Utility or controller classes handle complex data access shared by multiple routes.
- `OPTIONS` requests expose JSON schema for the current route.
- Extensibility is intentionally more constrained than traditional filter hooks to protect endpoint stability.

### Commerce Kit Implication

Commerce Kit's framework adapters should preserve this separation.

Recommended internal shape:

```ts
route -> use case -> schema/serializer -> response
```

Routes should not directly contain pricing, shipping, payment, or persistence logic. Schemas should be stable and versionable. Extension data should be added through namespaced schema extension points.

## Data Store Abstraction

WooCommerce's code reference exposes many data-store interfaces, including product, customer, coupon, order, order item, refund, payment token, shipping zone, and webhook data stores. HPOS relies on `WC_Data_Store::load('order')` returning either the legacy custom-post-type order data store or the custom order table data store depending on which storage mode is authoritative.

HPOS demonstrates why a storage abstraction matters. WooCommerce moved from WordPress posts/postmeta to ecommerce-specific order tables while preserving higher-level CRUD APIs.

### Commerce Kit Implication

Commerce Kit's adapter system should not be a thin SQL query bag. It should expose domain repositories and persistence capabilities.

Recommended domains:

- products
- variants
- prices
- carts
- orders
- order items
- payments
- refunds
- customers
- shipping zones
- webhooks
- idempotency records
- outbox/events

This makes it possible to support multiple persistence backends and future schema changes without rewriting core business logic.

## Order Storage And Migration Strategy

WooCommerce HPOS is a major architecture lesson. WooCommerce historically stored orders in WordPress `_posts` and `_postmeta`. HPOS introduced dedicated order tables:

- `_wc_orders`
- `_wc_order_addresses`
- `_wc_order_operational_data`
- `_wc_orders_meta`

HPOS uses the concept of authoritative and backup tables. The authoritative tables are used for normal reads/writes, and backup tables receive synchronized copies. Switching authoritative storage is blocked while orders are pending synchronization. Synchronization can be immediate, manual, or scheduled. Deletion synchronization is handled carefully to avoid accidental deletion of legitimate orders. Placeholder records reserve order IDs when synchronizing between storage models.

### Commerce Kit Implication

Commerce Kit should design persisted commerce data as durable, migration-sensitive infrastructure from the start.

Recommended decisions:

- Orders and payments should have dedicated tables, not generic metadata blobs.
- Storage adapters should expose migration state and capability checks.
- Schema changes should support backfill and compatibility windows.
- Core should include idempotent background migration primitives.
- Destructive migration behavior should be explicit and guarded.
- Order IDs should be stable across migration and provider synchronization.

Commerce Kit does not need HPOS-style dual-write on day one, but the architecture should not prevent it later.

## Order State Transitions

WooCommerce order/payment docs emphasize using domain methods for state transitions. Payment gateways should call `payment_complete()` rather than manually reducing stock and setting statuses. WooCommerce then handles status, stock, and actions through core.

Order status guidance includes:

- Use `on-hold` when payment needs manual verification.
- Use `failed` when payment fails after an order exists.
- Let core choose `processing` or `completed` when payment is complete.
- Add order notes for debugging and merchant visibility.

### Commerce Kit Implication

Commerce Kit should centralize order state transitions.

```ts
await commerce.orders.transition(orderId, {
  type: "payment.completed",
  paymentId,
  idempotencyKey,
})
```

Adapters and plugins should request transitions, not mutate order state directly. Core should apply validation, emit events, adjust stock, write audit notes, and enqueue follow-up jobs.

## Settings And Merchant Configuration

WooCommerce gateways and shipping methods define settings fields through WooCommerce's Settings API. The same class owns field definitions, saved option loading, and admin persistence hooks.

This is convenient for plugins, but it couples runtime provider code to admin settings UI and WordPress persistence.

### Commerce Kit Implication

Commerce Kit should learn from the need, not copy the coupling. Plugins and adapters should be able to declare configuration schema, but rendering and persistence should be owned by the host app or Commerce Kit admin layer.

Recommended model:

```ts
definePaymentAdapter({
  id: "stripe",
  configSchema: z.object({
    secretKey: z.string(),
    webhookSecret: z.string(),
    captureMode: z.enum(["automatic", "manual"]),
  }),
})
```

This supports validation, generated admin forms, environment-variable loading, and typed adapter config.

## Compatibility And Marketplace Discipline

WooCommerce extension best practices emphasize interoperability:

- Single core purpose.
- Use WooCommerce features as much as possible.
- Namespacing and prefixing to avoid conflicts.
- Check for existing declarations and dependencies.
- Avoid god objects.
- Separate business logic and presentation logic.
- Avoid direct access to internal WooCommerce code.
- Avoid custom database tables when platform-native storage is sufficient.
- Use official logging and debugging tools.
- Test against WordPress, WooCommerce, PHP, and active extension combinations.

### Commerce Kit Implication

Commerce Kit should eventually define plugin quality rules before a marketplace exists.

Recommended plugin rules:

- Declare capabilities and dependencies.
- Declare schema changes explicitly.
- Never mutate core tables outside adapter/repository APIs.
- Namespace all extension data.
- Keep UI extensions separate from domain hooks.
- Provide test fixtures for core flows touched by the plugin.
- Avoid importing internal modules.
- Provide migration and uninstall behavior for persisted data.

## Webhooks

WooCommerce supports webhooks through REST API resources and code-level webhook classes/data stores. REST API docs include webhook topics, delivery/payload, logging, visual interface, CRUD operations, and batch updates.

WooCommerce payment gateway docs also show provider callbacks through WC-API hooks, such as PayPal IPN.

### Commerce Kit Implication

Commerce Kit needs two webhook concepts:

- Provider webhooks received by Commerce Kit from payment, shipping, tax, or fulfillment providers.
- App webhooks emitted by Commerce Kit to downstream systems.

Provider webhooks should be verified, normalized, idempotently processed, and mapped into domain events. App webhooks should be delivered with retry, signing, logging, and replay support.

## Logging And Debugging

WooCommerce recommends using `WC_Logger`, making logging opt-in where relevant, and exposing logs through system status/debug tooling. The REST API also includes system status and system status tools endpoints.

### Commerce Kit Implication

Commerce Kit should expose structured logs and status diagnostics.

Useful diagnostics:

- adapter availability
- database migration status
- webhook verification status
- pending background jobs
- failed outbox events
- payment provider health
- shipping/tax adapter capability reports

This should be part of the architecture, not an afterthought.

## Architecture Strengths

- Mature plugin lifecycle model with activation, initialization, deactivation, and uninstall concepts.
- Hook/filter ecosystem makes many flows extensible without editing core.
- Payment and shipping provider contracts are documented as first-class extension APIs.
- Storefront API and admin REST API are separated by data sensitivity and use case.
- Data-store abstractions allowed WooCommerce to introduce HPOS without abandoning higher-level order APIs.
- Order state transitions are centralized through domain methods such as `payment_complete()`.
- Extension compatibility rules are explicit and marketplace-aware.
- Logging, system status, CLI, and migration tools are treated as operational features.

## Architecture Weaknesses

- WordPress hooks are stringly typed and can be hard to reason about at scale.
- Singleton/global patterns are pragmatic in WordPress but not ideal for a modern TypeScript library.
- Runtime behavior can depend on load order and whether a gateway or extension class has been loaded.
- Admin UI, settings persistence, and runtime provider behavior can become tightly coupled.
- Legacy compatibility creates substantial complexity, especially around storage migration.
- Monetary amounts in the REST API are decimal strings, which are JSON-safe but require careful internal conversion.

## Commerce Kit Priority Takeaways

- Define plugin lifecycle hooks before building a plugin ecosystem.
- Keep extension points typed, explicit, and namespaced rather than generic string filters.
- Make payment and shipping adapters capability-driven and state-machine-aware.
- Centralize order state transitions in core.
- Separate storefront-safe APIs from admin APIs.
- Separate routes, use cases, schemas, and persistence.
- Treat data-store abstraction as a domain repository layer, not a low-level SQL escape hatch.
- Design order/payment storage with future migrations in mind.
- Provide compatibility rules for plugins, including no internal imports and explicit schema changes.
- Add operational diagnostics for migrations, webhooks, adapters, logs, and background jobs.
