# Competitor API DX Research

## Purpose

This document summarizes ecommerce and commerce-adjacent SDK API patterns relevant to Commerce Kit. It focuses on developer experience, concrete API shapes, extension points, typing strategy, checkout/payment flow, and lessons for a Better Auth-inspired TypeScript commerce SDK.

Commerce Kit's current direction is a library-first, TypeScript-native commerce engine with one server config, adapters, plugins, framework handlers, generated schema, and an inferred client. The strongest external references are Spree for storefront client shape, Better Auth for configuration and plugin architecture, Stripe for resource ergonomics and webhooks, Medusa for headless commerce modules, Vendure for plugin extensibility, Saleor for API contracts, and commercetools for generated type coverage.

## Evaluation Criteria

- Setup simplicity: how much code is needed before the first useful call.
- Domain vocabulary: whether APIs model commerce concepts directly.
- Type safety: whether inputs, outputs, optional capabilities, and extensions are represented in TypeScript.
- Extensibility: whether custom domains, integrations, and provider-specific behavior are first-class.
- Checkout flow: how cart, payment, delivery, and order completion are represented.
- Error handling: whether failures are structured and predictable.
- Webhook handling: whether raw-body verification and idempotency are explicit.
- Escape hatches: whether custom endpoints or low-level requests remain possible without abandoning SDK behavior.

## Spree SDK

### Positioning

Spree's Store SDK is the most directly relevant reference for Commerce Kit's client API shape. It exposes storefront-focused domain namespaces over Spree's Store API and keeps the default interaction model simple.

### Setup API

```ts
import { createClient } from "@spree/sdk"

const client = createClient({
  baseUrl: "https://api.mystore.com",
  publishableKey: "pk_xxx",
})
```

### Top-Level API Shape

```ts
client.products.list(...)
client.products.get(...)

client.categories.list(...)
client.categories.get(...)
client.categories.products.list(...)

client.carts.create(...)
client.carts.get(...)
client.carts.update(...)
client.carts.complete(...)

client.carts.items.create(...)
client.carts.items.update(...)
client.carts.items.delete(...)

client.carts.paymentSessions.create(...)
client.carts.paymentSessions.complete(...)

client.orders.get(...)
client.customer.orders.list(...)

client.markets.list(...)
client.markets.resolve(...)
client.wishlists.items.create(...)
```

Spree's nested-resource convention is clear:

```ts
client.parent.nested.method(parentId, ...)
```

Examples:

```ts
await client.carts.items.create(cartId, params, options)
await client.carts.fulfillments.update(cartId, fulfillmentId, params, options)
await client.categories.products.list(categoryId, params, options)
await client.wishlists.items.create(wishlistId, params, options)
```

### Products and Categories

Spree supports list/get, search, filtering, sorting, field selection, association expansion, nested expansion, product filters, categories, category products, and geography.

```ts
const products = await client.products.list({
  page: 1,
  limit: 25,
  expand: ["variants", "media", "categories"],
})
```

Filtering uses flat Ransack-style keys. The SDK wraps them into API query params.

```ts
const products = await client.products.list({
  name_cont: "shirt",
  price_gte: 20,
  price_lte: 100,
  with_option_value_ids: ["optval_abc", "optval_def"],
  in_stock: true,
  search: "blue shirt",
})
```

Sorting uses string values, with `-` for descending order.

```ts
const products = await client.products.list({
  sort: "-price",
})
```

Field selection reduces payload size.

```ts
const product = await client.products.get("spree-tote", {
  fields: ["name", "slug", "price"],
  expand: ["media"],
})
```

Nested expansion uses dot notation.

```ts
const product = await client.products.get("spree-tote", {
  expand: ["variants.media", "option_types"],
})
```

Spree also exposes a product filters endpoint for faceted storefront UIs.

```ts
const filters = await client.products.filters({
  category_id: "ctg_abc123",
})
```

The response includes available filters, sort options, default sort, and total count. This is valuable because faceted search UIs otherwise require duplicated query logic in the application.

Categories include list/get plus nested category products.

```ts
const categories = await client.categories.list({
  depth_eq: 1,
})

const category = await client.categories.get("clothing/shirts", {
  expand: ["ancestors", "children"],
})

const categoryProducts = await client.categories.products.list("clothing/shirts", {
  page: 1,
  limit: 12,
  expand: ["media", "default_variant"],
})
```

Notable product detail: Spree supports `prior_price` expansion for EU Omnibus Directive requirements.

```ts
const product = await client.products.get("spree-tote", {
  expand: ["prior_price"],
})
```

### Cart, Checkout, and Orders

Spree separates active carts from completed orders.

```ts
const cart = await client.carts.create()

const cart = await client.carts.get(cartId, {
  spreeToken: cart.token,
})

await client.carts.delete(cartId, {
  spreeToken: cart.token,
})
```

Guest cart access uses `spreeToken`. Authenticated customer access uses JWT `token`.

```ts
await client.carts.associate(cartId, {
  token: jwtToken,
  spreeToken: cart.token,
})
```

Cart line items are nested under `carts.items`.

```ts
const options = { spreeToken: cart.token }

await client.carts.items.create(cartId, {
  variant_id: "var_123",
  quantity: 2,
}, options)

await client.carts.items.update(cartId, lineItemId, {
  quantity: 3,
}, options)

await client.carts.items.delete(cartId, lineItemId, options)
```

Discount codes, gift cards, and store credits are nested action resources.

```ts
await client.carts.discountCodes.apply(cartId, "SAVE20", options)
await client.carts.discountCodes.remove(cartId, "SAVE20", options)

await client.carts.giftCards.apply(cartId, "GC-ABCD-1234", options)
await client.carts.giftCards.remove(cartId, "gc_abc123", options)

await client.carts.storeCredits.apply(cartId, undefined, options)
await client.carts.storeCredits.apply(cartId, 25.00, options)
await client.carts.storeCredits.remove(cartId, options)
```

Checkout information is updated through the cart.

```ts
await client.carts.update(cartId, {
  email: "customer@example.com",
  shipping_address: {
    first_name: "John",
    last_name: "Doe",
    address1: "123 Main St",
    city: "New York",
    postal_code: "10001",
    phone: "+1 555 123 4567",
    country_iso: "US",
    state_abbr: "NY",
  },
  billing_address_id: "addr_xxx",
}, { spreeToken: cart.token })
```

Completion turns a cart into an order.

```ts
await client.carts.complete(cartId, {
  spreeToken: cart.token,
})
```

Completed orders are retrieved separately.

```ts
const order = await client.orders.get("R123456789", {
  expand: ["items", "fulfillments"],
}, { spreeToken: cart.token })

const orders = await client.customer.orders.list({}, { token })
```

### Payments and Delivery

Spree exposes payment methods, payments, and fulfillments through the cart response rather than always separate list endpoints.

```ts
const cart = await client.carts.get(cartId, options)
const paymentMethods = cart.payment_methods
const payments = cart.payments
const fulfillments = cart.fulfillments
```

Payment methods include `session_required`.

```ts
if (paymentMethod.session_required) {
  // use paymentSessions
} else {
  // use payments.create
}
```

Direct payments are for non-session methods.

```ts
const payment = await client.carts.payments.create(cartId, {
  payment_method_id: "pm_xxx",
  amount: "99.99",
  metadata: {
    purchase_order_number: "PO-12345",
  },
}, options)
```

Payment sessions provide a unified provider-agnostic interface for gateways such as Stripe, Adyen, and PayPal.

```ts
const session = await client.carts.paymentSessions.create(cartId, {
  payment_method_id: "pm_xxx",
  amount: "99.99",
  external_data: {
    return_url: "https://mystore.com/checkout/complete",
  },
}, options)

const existing = await client.carts.paymentSessions.get(cartId, session.id, options)

await client.carts.paymentSessions.update(cartId, session.id, {
  amount: "149.99",
}, options)

const completed = await client.carts.paymentSessions.complete(
  cartId,
  session.id,
  { session_result: "success" },
  options,
)
```

Delivery rate selection happens through fulfillments.

```ts
await client.carts.fulfillments.update(cartId, fulfillmentId, {
  selected_delivery_rate_id: "rate_xxx",
}, options)
```

### Markets

Spree models markets directly. This is useful for country, currency, locale, and tax-inclusive behavior.

```ts
const { data: markets } = await client.markets.list()

const market = await client.markets.get("mkt_gbHJdmfrXB")

const market = await client.markets.resolve("DE")

const { data: countries } = await client.markets.countries.list("mkt_gbHJdmfrXB")

const country = await client.markets.countries.get("mkt_gbHJdmfrXB", "US", {
  expand: ["states"],
})
```

Market shape includes currency, default locale, supported locales, tax-inclusive flag, default marker, and countries.

### Customer Account

Customer APIs are grouped under `client.customer` rather than generic `customers`.

```ts
const options = { token: jwtToken }

const profile = await client.customer.get(options)

await client.customer.update({
  first_name: "John",
  last_name: "Doe",
}, options)
```

Nested account resources include addresses, store credits, and credit cards.

```ts
const { data: addresses } = await client.customer.addresses.list({}, options)
const address = await client.customer.addresses.get("addr_xxx", options)

await client.customer.addresses.create({
  first_name: "John",
  last_name: "Doe",
  address1: "123 Main St",
  city: "New York",
  postal_code: "10001",
  country_iso: "US",
  state_abbr: "NY",
  is_default_billing: true,
  is_default_shipping: true,
}, options)

await client.customer.addresses.update("addr_xxx", {
  city: "Brooklyn",
  is_default_billing: true,
}, options)

await client.customer.addresses.delete("addr_xxx", options)
```

Saved payment and credit resources are read-oriented.

```ts
const { data: credits } = await client.customer.storeCredits.list({}, options)
const credit = await client.customer.storeCredits.get("sc_xxx", options)

const { data: cards } = await client.customer.creditCards.list({}, options)
const card = await client.customer.creditCards.get("cc_xxx", options)
await client.customer.creditCards.delete("cc_xxx", options)
```

### Wishlists

Wishlists are a clean example of an optional feature that Commerce Kit should likely model as a plugin-gated namespace.

```ts
const options = { token: jwtToken }

const { data: wishlists } = await client.wishlists.list({}, options)

const wishlist = await client.wishlists.get("wl_xxx", {
  expand: ["wished_items"],
}, options)

const newWishlist = await client.wishlists.create({
  name: "Birthday Ideas",
  is_private: true,
}, options)

await client.wishlists.update("wl_xxx", {
  name: "Updated Name",
}, options)

await client.wishlists.delete("wl_xxx", options)
```

Wishlist items follow the nested resource convention.

```ts
await client.wishlists.items.create("wl_xxx", {
  variant_id: "var_123",
  quantity: 1,
}, options)

await client.wishlists.items.update("wl_xxx", "wi_xxx", {
  quantity: 2,
}, options)

await client.wishlists.items.delete("wl_xxx", "wi_xxx", options)
```

### Authentication

Spree has three practical modes:

- Publishable key for public browsing.
- JWT token for authenticated customer operations.
- Guest cart token for guest checkout.

```ts
const { token, user } = await client.auth.login({
  email: "customer@example.com",
  password: "password123",
})

const newTokens = await client.auth.refresh({ token })

const { token, user } = await client.customers.create({
  email: "new@example.com",
  password: "password123",
  password_confirmation: "password123",
  first_name: "John",
  last_name: "Doe",
})
```

### Localization and Request Options

Locale, currency, and country are passed per request.

```ts
const products = await client.products.list({}, {
  locale: "fr",
  currency: "EUR",
  country: "FR",
})
```

### Error Handling

Spree exposes a structured `SpreeError`.

```ts
import { SpreeError } from "@spree/sdk"

try {
  await client.products.get("non-existent")
} catch (error) {
  if (error instanceof SpreeError) {
    console.log(error.code)
    console.log(error.message)
    console.log(error.status)
    console.log(error.details)
  }
}
```

### TypeScript and Runtime Validation

Spree exports generated TypeScript interfaces and Zod schemas.

```ts
import type {
  Product,
  Order,
  Cart,
  Variant,
  Category,
  LineItem,
  Address,
  Customer,
  PaginatedResponse,
} from "@spree/sdk"
```

Customization uses declaration merging for TypeScript and `.extend()` for Zod schemas.

```ts
declare module "@spree/sdk" {
  interface Product {
    brand_id: string
  }
}
```

```ts
import { ProductSchema } from "@spree/sdk/zod"

const MyProductSchema = ProductSchema.extend({
  brand_id: z.string(),
})
```

### Escape Hatch

Spree exposes `client.request<T>()` for custom Store API endpoints while preserving auth headers, retry behavior, and locale/currency defaults.

```ts
interface Brand {
  id: string
  name: string
  slug: string | null
}

const brands = await client.request<PaginatedResponse<Brand>>("GET", "/brands")
const nike = await client.request<Brand>("GET", "/brands/nike")
```

### Strengths

- Very clear storefront API vocabulary.
- Nested-resource API works well for cart, category products, customer resources, and wishlists.
- Cart and completed order separation is intuitive.
- Payment sessions are a strong provider-neutral abstraction.
- Markets are first-class and useful for real global commerce.
- `expand`, nested expand, and `fields` are high-value storefront optimizations.
- Product filters endpoint directly supports faceted search UX.
- Structured errors and generated types improve DX.
- `request<T>()` is a practical escape hatch.

### Weaknesses

- Flat filter keys such as `name_cont` and `price_gte` are concise but less discoverable and less type-safe than structured query objects.
- Money values are decimal strings; this avoids transport float errors but pushes calculation responsibility to app code.
- Per-call `{ token }` and `{ spreeToken }` options are explicit but repetitive.
- Declaration merging is pragmatic, but it is less safe than server-config-derived plugin inference.
- Store SDK is tied to a backend platform; it does not solve library-as-engine composition.

### Lessons for Commerce Kit

- Borrow the domain namespace shape, especially nested resources.
- Borrow `paymentSessions` as a first-class checkout concept.
- Borrow `expand`, nested expand, `fields`, and product filter endpoints.
- Borrow `markets.resolve(country)` if markets are included in v1 or a first-party plugin.
- Prefer structured typed filters over Ransack-style flat keys.
- Prefer integer minor units internally and across the SDK rather than decimal strings.
- Prefer config-derived plugin/client inference over declaration merging for first-party extension surfaces.
- Keep a typed `request<T>()` escape hatch.

## Better Auth

### Positioning

Better Auth is not an ecommerce SDK, but it is the strongest reference for Commerce Kit's intended architecture and developer experience. The relevant ideas are one canonical server config, first-class plugins, framework route handlers, adapters, CLI, generated schema, and client extensions inferred from server plugins.

### Server Setup

```ts
import { betterAuth } from "better-auth"

export const auth = betterAuth({
  database: ..., 
  plugins: [
    ...
  ],
})
```

### Client Setup

```ts
import { createAuthClient } from "better-auth/client"

export const authClient = createAuthClient({
  plugins: [
    ...
  ],
})
```

### Plugin API

Better Auth plugins can define endpoints, schema, hooks, middleware, rate limits, trusted origins, request/response hooks, and client plugins.

```ts
import type { BetterAuthPlugin } from "better-auth"

export const myPlugin = () => {
  return {
    id: "my-plugin",
  } satisfies BetterAuthPlugin
}
```

Server endpoints are defined declaratively.

```ts
import { createAuthEndpoint } from "better-auth/api"

const myPlugin = () => ({
  id: "my-plugin",
  endpoints: {
    getHelloWorld: createAuthEndpoint(
      "/my-plugin/hello-world",
      { method: "GET" },
      async (ctx) => {
        return ctx.json({ message: "Hello World" })
      },
    ),
  },
} satisfies BetterAuthPlugin)
```

Plugin schema can add tables or extend core tables.

```ts
const myPlugin = () => ({
  id: "my-plugin",
  schema: {
    myTable: {
      fields: {
        name: { type: "string" },
      },
    },
  },
} satisfies BetterAuthPlugin)
```

Client plugins can infer server plugin endpoints.

```ts
import type { BetterAuthClientPlugin } from "better-auth/client"
import type { myPlugin } from "./plugin"

const myPluginClient = () => ({
  id: "my-plugin",
  $InferServerPlugin: {} as ReturnType<typeof myPlugin>,
} satisfies BetterAuthClientPlugin)
```

Client plugins can add actions and atoms.

```ts
const myPluginClient = {
  id: "my-plugin",
  getActions: ($fetch) => ({
    myCustomAction: async (data, fetchOptions) => {
      return $fetch("/custom/action", {
        method: "POST",
        body: data,
        ...fetchOptions,
      })
    },
  }),
} satisfies BetterAuthClientPlugin
```

### Strengths

- One obvious config object.
- Plugins are the main feature delivery mechanism.
- Server plugins and client plugins are paired but separate.
- Framework route handlers are thin wrappers.
- Schema generation and migrations reduce setup friction.
- Type propagation makes plugin installation visible in app code.

### Weaknesses

- Plugin authors must understand several extension surfaces.
- Client/server plugin pairing can add ceremony.
- Endpoint-based plugin inference maps well to auth but needs careful adaptation for commerce workflows.

### Lessons for Commerce Kit

- `createCommerce()` should be the single source of truth for runtime, schema, routes, plugins, adapters, and client typing.
- Commerce plugins should be able to contribute schema, hooks, server APIs, client APIs, order states, routes, and webhooks.
- Commerce Kit should expose framework-specific adapters, not framework-specific domain logic.
- CLI should exist early because schema generation and plugin installation are central to the DX.

## Stripe Node SDK

### Positioning

Stripe is the strongest reference for resource-oriented API ergonomics, error classes, idempotency, retries, and webhook verification.

### Setup API

```ts
import Stripe from "stripe"

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2025-...",
  maxNetworkRetries: 2,
  timeout: 10000,
})
```

### Resource API Shape

```ts
await stripe.customers.create(...)
await stripe.paymentIntents.create(...)
await stripe.paymentIntents.retrieve(...)
await stripe.refunds.create(...)
```

### Webhooks

Stripe makes signature verification explicit and requires raw request body preservation.

```ts
const event = stripe.webhooks.constructEvent(
  rawBody,
  signature,
  endpointSecret,
)
```

### Strengths

- Simple constructor.
- Predictable resource namespaces.
- Strong docs and error taxonomy.
- Webhook verification is explicit and hard to confuse with normal JSON parsing.
- Built-in retry and timeout options.
- Good idempotency-key support for writes.

### Weaknesses

- API version/type mismatch can require workarounds.
- `expand` can produce types that are difficult to narrow.
- Secret-key SDK is server-only; storefront usage requires separate patterns.

### Lessons for Commerce Kit

- Commerce Kit should expose structured error classes.
- Webhook helpers should require raw body handling and make signature verification explicit.
- Payment, shipping, delivery, and payout adapters should support idempotency keys where provider APIs allow them.
- SDK request options should include retry, timeout, headers, and idempotency without requiring custom middleware.

## Medusa JS SDK

### Positioning

Medusa is a headless commerce platform with a JS SDK for storefronts, admin customizations, and custom routes.

### Setup API

```ts
import Medusa from "@medusajs/js-sdk"

export const sdk = new Medusa({
  baseUrl: process.env.NEXT_PUBLIC_MEDUSA_BACKEND_URL!,
  debug: process.env.NODE_ENV === "development",
  publishableKey: process.env.NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY,
})
```

Admin session usage:

```ts
export const sdk = new Medusa({
  baseUrl: "/",
  debug: import.meta.env.DEV,
  auth: {
    type: "session",
  },
})
```

### API Shape

```ts
await sdk.store.product.list(...)
await sdk.admin.product.list(...)
await sdk.admin.product.update(...)
```

### Custom Routes

```ts
await sdk.client.fetch("/custom")

await sdk.client.fetch("/custom", {
  method: "post",
  body: {
    id: "123",
  },
})
```

### Streaming

Medusa exposes `fetchStream` for server-sent events.

```ts
const { stream, abort } = await sdk.client.fetchStream("/admin/stream")

for await (const chunk of stream) {
  // consume chunk
}
```

### Errors

Medusa request errors use `FetchError` with status, status text, and message.

```ts
try {
  await sdk.store.customer.listAddress()
} catch (error) {
  const fetchError = error as FetchError
  console.log(fetchError.status)
  console.log(fetchError.statusText)
}
```

### Strengths

- Simple client constructor.
- Store/admin separation is explicit.
- Custom route escape hatch preserves SDK auth and headers.
- Strong integration with TanStack Query examples.
- Supports framework-specific cache metadata such as Next.js tags through request options.

### Weaknesses

- Platform runtime assumptions are heavier than Commerce Kit's library goal.
- SDK shape is coupled to Medusa backend concepts.
- Extension model is powerful but broader than a small library needs.

### Lessons for Commerce Kit

- Consider separate client entrypoints or namespaces for storefront and admin/server use if the API surfaces diverge.
- Provide a custom fetch/request escape hatch.
- Support framework request options without hard-coding one framework into core.
- Avoid platform-level runtime assumptions in the base package.

## Vendure

### Positioning

Vendure is a TypeScript/NestJS commerce framework with a strong plugin system and GraphQL APIs.

### Config and Plugin Model

Vendure applications are configured through `VendureConfig`. Plugins can add entities, services, admin UI extensions, GraphQL schema extensions, resolvers, and lifecycle behavior.

Conceptual shape:

```ts
export const config: VendureConfig = {
  apiOptions: {
    port: 3000,
    adminApiPath: "admin-api",
    shopApiPath: "shop-api",
  },
  plugins: [
    ...
  ],
}
```

### API Style

Vendure is GraphQL-first. Storefront clients typically use GraphQL clients and code generation rather than a single domain-method SDK.

### Strengths

- Plugin architecture is mature and powerful.
- Extensions can add deep platform behavior.
- GraphQL gives precise selection sets and generated types.
- Admin and shop APIs are distinct.

### Weaknesses

- NestJS and GraphQL add adoption weight.
- Storefront API calls often require GraphQL client/codegen setup.
- More platform-like than library-like.

### Lessons for Commerce Kit

- Borrow the seriousness of plugin boundaries, not the framework weight.
- Keep admin/storefront distinction possible without requiring GraphQL.
- Make extension points typed, ordered, and explicit.

## Saleor

### Positioning

Saleor is API-first and GraphQL-native. The relevant DX lesson is the strength of a stable typed API contract plus app/webhook extension model.

### API Style

Saleor clients generally use GraphQL operations, generated types, auth SDKs, and app SDKs. Extensions are often external apps and webhook consumers rather than in-process plugins.

### Strengths

- Strong API contract.
- Generated TypeScript types can be excellent.
- App/webhook model works well for external integrations.
- Clear storefront/backend separation.

### Weaknesses

- GraphQL/codegen adds ceremony.
- Extension behavior is distributed across external apps, webhooks, and API usage rather than one in-process SDK surface.
- Less relevant to Commerce Kit's library-in-your-stack goal.

### Lessons for Commerce Kit

- Preserve a stable API contract and generated/inferred types.
- Provide webhook-first integration patterns.
- Do not require GraphQL/codegen for the default developer path.

## commercetools TypeScript SDK

### Positioning

commercetools is the strongest reference for generated type coverage and enterprise-grade API composition, but its SDK is more verbose than Commerce Kit should be.

### Setup API

```ts
import {
  ClientBuilder,
  type AuthMiddlewareOptions,
  type HttpMiddlewareOptions,
} from "@commercetools/ts-client"

const authMiddlewareOptions: AuthMiddlewareOptions = {
  host: "https://auth.{region}.commercetools.com",
  projectKey,
  credentials: {
    clientId: "{clientID}",
    clientSecret: "{clientSecret}",
  },
  scopes,
  httpClient: fetch,
}

const httpMiddlewareOptions: HttpMiddlewareOptions = {
  host: "https://api.{region}.commercetools.com",
  httpClient: fetch,
}

const client = new ClientBuilder()
  .withProjectKey(projectKey)
  .withClientCredentialsFlow(authMiddlewareOptions)
  .withHttpMiddleware(httpMiddlewareOptions)
  .withLoggerMiddleware()
  .build()
```

### API Builder Shape

```ts
import { createApiBuilderFromCtpClient } from "@commercetools/platform-sdk"

const apiRoot = createApiBuilderFromCtpClient(client).withProjectKey({
  projectKey,
})

const response = await apiRoot
  .shoppingLists()
  .withId({ ID })
  .get()
  .execute()
```

List calls use `queryArgs`.

```ts
const response = await apiRoot.shoppingLists().get({
  queryArgs: {
    where: ["lineItems is not empty"],
    sort: "name.en-US desc",
    expand: ["customer"],
    limit: 5,
    offset: 0,
  },
}).execute()
```

Updates require versioned action arrays.

```ts
const update = {
  version: 1,
  actions: [
    { action: "setKey", key: "new-key" },
  ],
}

await apiRoot
  .shoppingLists()
  .withId({ ID })
  .post({ body: update })
  .execute()
```

### Strengths

- Excellent generated API coverage.
- Middleware system handles auth, HTTP, logging, retry, and customization.
- Versioned updates are safe for concurrent enterprise workflows.
- API-specific packages keep large domains separated.

### Weaknesses

- Setup is verbose.
- Builder chains are powerful but heavy.
- Query predicates and action arrays are less approachable for small teams.
- Too much ceremony for Commerce Kit's intended first-run DX.

### Lessons for Commerce Kit

- Borrow generated type rigor and clear package boundaries.
- Avoid builder-chain ceremony in the primary API.
- Consider optimistic concurrency/version fields where order/catalog updates need safety, but do not force every update into action arrays unless required.

## Shopify SDKs

### Positioning

Shopify is a reference for multi-runtime adapters, typed GraphQL clients, OAuth/session complexity, and webhook handling.

### Setup Pattern

Shopify SDK setup typically configures app keys, scopes, host name, API version, runtime adapter, session storage, and webhooks.

Conceptual shape:

```ts
const shopify = shopifyApi({
  apiKey: process.env.SHOPIFY_API_KEY!,
  apiSecretKey: process.env.SHOPIFY_API_SECRET!,
  scopes: ["read_products", "write_orders"],
  hostName: process.env.HOST!,
  apiVersion: "...",
})
```

### Strengths

- Serious treatment of OAuth, sessions, API versions, and webhooks.
- Strong GraphQL type story when paired with codegen/client tooling.
- Multi-runtime support is explicit.

### Weaknesses

- OAuth and session setup are complex.
- REST plus GraphQL creates split mental models.
- Codegen and body-parser details can create onboarding friction.

### Lessons for Commerce Kit

- Make runtime/framework adapter boundaries explicit.
- Keep webhook raw-body handling documented and helper-driven.
- Do not impose OAuth/session complexity; Commerce Kit should remain auth-agnostic and accept request context from the app.

## API Design Recommendations for Commerce Kit

### Server Config

Commerce Kit should keep the current Better Auth-inspired singleton config direction.

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { stripePayments } from "@commerce-kit/stripe"
import { couponsPlugin } from "@commerce-kit/coupons"
import { wishlistsPlugin } from "@commerce-kit/wishlists"
import { marketsPlugin } from "@commerce-kit/markets"

export const commerce = createCommerce({
  database: drizzleAdapter({ db, provider: "postgres" }),
  payments: [
    stripePayments({
      adapterId: "stripe-main",
      secretKey: env.STRIPE_SECRET_KEY,
      webhookSecret: env.STRIPE_WEBHOOK_SECRET,
    }),
  ],
  plugins: [
    couponsPlugin(),
    wishlistsPlugin(),
    marketsPlugin(),
  ],
})
```

### Client Config

The client should infer API surface from the server instance type.

```ts
import { createCommerceClient } from "@commerce-kit/client"
import type { commerce } from "@/lib/commerce"

export const client = createCommerceClient<typeof commerce>({
  baseUrl: "/api/commerce",
})
```

For public storefront usage, request context should come from the framework route and application auth, not from Commerce Kit owning auth.

### Core Namespace Shape

Recommended baseline:

```ts
client.products.list(...)
client.products.get(...)
client.products.create(...)
client.products.update(...)
client.products.archive(...)
client.products.versions(...)
client.products.getVersion(...)

client.categories.list(...)
client.categories.get(...)
client.categories.create(...)
client.categories.update(...)
client.categories.archive(...)
client.categories.products.list(...)

client.carts.get(...)
client.carts.create(...)
client.carts.update(...)
client.carts.delete(...)
client.carts.items.create(...)
client.carts.items.update(...)
client.carts.items.delete(...)

client.orders.list(...)
client.orders.get(...)
client.orders.create(...)
client.orders.calculate(...)
client.orders.transition(...)

client.payments.list(...)
client.payments.get(...)
```

Optional/plugin-gated namespaces:

```ts
client.wishlists.list(...)
client.wishlists.items.create(...)

client.markets.list(...)
client.markets.resolve(...)

client.shipping.rates(...)
client.delivery.quote(...)

client.marketplace.vendors.list(...)
client.marketplace.vendorOrders.transition(...)
```

### Query API

Commerce Kit should prefer structured typed query inputs over flat Ransack-style keys.

```ts
await client.products.list({
  where: {
    title: { contains: "shirt" },
    price: { gte: 2000, lte: 10000 },
    status: "active",
    optionValueIds: { in: ["optval_abc", "optval_def"] },
    inventory: { gt: 0 },
  },
  orderBy: [
    { price: "asc" },
    { createdAt: "desc" },
  ],
  limit: 25,
  cursor: "next_cursor",
  expand: ["variants", "media", "categories"],
  fields: ["id", "title", "slug", "price"],
})
```

`search` should exist only where full-text relevance is materially different from structured filtering.

```ts
await client.products.search({
  query: "blue shirt",
  where: {
    status: "active",
    inventory: { gt: 0 },
  },
  orderBy: [{ relevance: "desc" }],
  limit: 12,
})
```

### Expand and Field Selection

Commerce Kit should support Spree-style `expand` and `fields`, but make them typed per namespace.

```ts
await client.products.get("prod_123", {
  expand: ["variants.media", "categories"],
  fields: ["id", "title", "slug", "price"],
})
```

Rules:

- `id` is always returned.
- Expanded relations are included even if omitted from `fields`.
- Nested expand depth should be capped.
- Unsupported expand paths should fail at compile time where statically visible and at runtime otherwise.

### Cart and Checkout

Commerce Kit should treat cart as the active checkout aggregate.

```ts
const cart = await client.carts.create()

await client.carts.items.create(cart.id, {
  variantId: "var_123",
  quantity: 2,
})

await client.carts.update(cart.id, {
  email: "customer@example.com",
  shippingAddress: {
    firstName: "John",
    lastName: "Doe",
    address1: "123 Main St",
    city: "New York",
    postalCode: "10001",
    countryCode: "US",
    stateCode: "NY",
  },
})
```

Completion should be explicit and should return an order.

```ts
const order = await client.carts.complete(cart.id)
```

If Commerce Kit keeps `orders.create(...)` as the initial write API, it should still distinguish draft/active cart behavior from finalized order behavior. Orders are financial records and should not behave like mutable carts after confirmation.

### Payments and Payment Sessions

Commerce Kit should introduce payment sessions as a first-class checkout API, separate from the immutable payment ledger.

```ts
const session = await client.carts.paymentSessions.create(cart.id, {
  paymentAdapterId: "stripe-main",
  amount: { amount: 9999, currency: "USD" },
  externalData: {
    returnUrl: "https://mystore.com/checkout/complete",
  },
})

await client.carts.paymentSessions.update(cart.id, session.id, {
  amount: { amount: 14999, currency: "USD" },
})

await client.carts.paymentSessions.complete(cart.id, session.id, {
  result: "success",
})
```

Direct payment creation should be reserved for non-session methods or server-side/admin operations.

```ts
await client.carts.payments.create(cart.id, {
  paymentAdapterId: "manual",
  amount: { amount: 9999, currency: "USD" },
})
```

The public payment ledger should remain read-oriented.

```ts
await client.payments.list({ where: { orderId } })
await client.payments.get(paymentId)
```

### Markets

Markets should be a serious candidate for v1 or a first-party plugin because currency, locale, country, and tax-inclusive behavior affect catalog, checkout, and pricing.

```ts
await client.markets.list()
await client.markets.get("mkt_123")
await client.markets.resolve({ countryCode: "DE" })
await client.markets.countries.list("mkt_123")
await client.markets.countries.get("mkt_123", "US", {
  expand: ["states"],
})
```

Request options can carry market context.

```ts
await client.products.list({}, {
  marketId: "mkt_123",
  locale: "de",
  currency: "EUR",
  countryCode: "DE",
})
```

### Customer Account

Commerce Kit currently says auth is outside scope. It can still model customer records and account-owned commerce resources without owning authentication.

Recommended split:

- `customers` for server/admin customer management.
- `customer` or `me` for current authenticated customer storefront operations, if exposed by the framework route context.

```ts
await client.customers.list(...)
await client.customers.get("cust_123")
await client.customers.create(...)
await client.customers.update("cust_123", ...)

await client.customer.get()
await client.customer.addresses.list()
await client.customer.addresses.create(...)
await client.customer.orders.list(...)
```

The `customer` namespace should exist only for HTTP client surfaces where `resolveContext` can identify the current customer.

### Wishlists

Wishlists should be a plugin-gated namespace.

```ts
await client.wishlists.list()
await client.wishlists.get("wl_123", {
  expand: ["items.variant.product"],
})
await client.wishlists.create({
  name: "Birthday Ideas",
  private: true,
})
await client.wishlists.update("wl_123", {
  name: "Updated Name",
})
await client.wishlists.delete("wl_123")

await client.wishlists.items.create("wl_123", {
  variantId: "var_123",
  quantity: 1,
})
await client.wishlists.items.update("wl_123", "wi_123", {
  quantity: 2,
})
await client.wishlists.items.delete("wl_123", "wi_123")
```

If the plugin is not installed, `client.wishlists` should not exist at compile time.

### Money

Commerce Kit should not follow Spree's decimal string API for core calculations. The project already documents integer minor units, and that is the right direction.

Recommended input/output shape:

```ts
type Money = {
  amount: number
  currency: string
}
```

Example:

```ts
{
  total: { amount: 12999, currency: "USD" },
  displayTotal: "$129.99",
}
```

Rules:

- `amount` is always integer minor units.
- `currency` is ISO 4217.
- Display strings are optional convenience output and never calculation input.
- Cross-currency cart/order creation should fail or split before persistence.

### Errors

Commerce Kit should expose structured error classes inspired by Spree and Stripe.

```ts
try {
  await client.products.get("missing")
} catch (error) {
  if (error instanceof CommerceError) {
    console.log(error.code)
    console.log(error.status)
    console.log(error.message)
    console.log(error.details)
  }
}
```

Core error classes should align with the architecture docs:

- `CommerceConfigError`
- `CommercePluginError`
- `CommerceValidationError`
- `CommerceNotFoundError`
- `CommerceOrderError`
- `CommerceStateError`
- `CommercePaymentError`
- `CommerceMoneyError`
- `CommerceWebhookError`

### Webhooks

Commerce Kit should keep webhook handling separate from normal route handlers.

```ts
export async function POST(req: Request, { params }: { params: { adapterId: string } }) {
  return createWebhookHandler(commerce, {
    type: "payment",
    adapterId: params.adapterId,
  })(req)
}
```

Rules:

- Webhook handlers must preserve raw body bytes.
- Adapters verify signatures before core or plugins see events.
- Webhook event IDs must be persisted separately from transaction references.
- Plugin webhook handlers receive normalized verified events only.
- Idempotency must be enforced at the database boundary.

### Escape Hatch

The client should expose a low-level typed request API that preserves configured base URL, headers, auth/context forwarding, locale/currency/market options, retries, and errors.

```ts
const brands = await client.request<PaginatedResponse<Brand>>({
  method: "GET",
  path: "/brands",
})
```

This should be positioned as an escape hatch, not the primary way to use Commerce Kit.

### API Naming Rules

- Prefer commerce domain names over transport names.
- Prefer explicit workflow methods over generic CRUD for workflow operations.
- Omit unsupported methods instead of exposing methods that fail at runtime.
- Use nested resources when the child resource cannot be understood without a parent.
- Use plugin-gated namespaces for optional domains.
- Keep provider-specific fields under `externalData`, `providerData`, or adapter-owned types.
- Avoid exposing raw ORM/query-builder concepts in the public client.

## Priority Recommendations

1. Keep `createCommerce()` as the only source of truth for server, schema, route, plugin, adapter, and client types.
2. Preserve `commerce.domain.method(...)` for both in-process server usage and HTTP client usage.
3. Add Spree-style nested resource conventions for carts, categories, customer account resources, and wishlists.
4. Add first-class `paymentSessions` separate from the immutable payment ledger.
5. Add typed `expand`, nested expand, and `fields` to product/category/order reads.
6. Prefer structured typed `where` filters over flat string-key filters.
7. Treat markets as either v1 core or a first-party plugin because market context affects pricing, tax, locale, country, and checkout.
8. Keep money as integer minor units throughout the SDK.
9. Provide structured error classes and stable error codes.
10. Provide a typed `request<T>()` escape hatch.
11. Keep webhooks raw-body-first, adapter-verified, idempotent, and separate from normal HTTP routes.
12. Make optional domains disappear from the inferred client when their plugin or adapter is not installed.
