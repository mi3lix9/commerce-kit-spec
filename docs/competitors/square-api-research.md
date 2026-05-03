# Square API Research

## Purpose

This document captures API and SDK design lessons from Square's TypeScript SDK and API reference. Square is less directly comparable to Commerce Kit than Spree or Medusa because it is a payments and seller-platform API rather than a commerce engine, but it is a strong reference for financial API rigor.

The most relevant Square lessons for Commerce Kit are idempotent mutations, explicit money representation, request-level operational controls, consistent pagination, raw response escape hatches, typed errors, webhook signature verification, and clear separation between orders, payments, refunds, catalog, checkout links, customers, and webhook subscriptions.

## Sources Reviewed

- Square TypeScript SDK README: `https://github.com/square/square-nodejs-sdk/blob/HEAD/README.md`
- Square TypeScript SDK reference: `https://github.com/square/square-nodejs-sdk/blob/HEAD/reference.md`

## SDK Setup

Square's current TypeScript SDK exposes a generated `SquareClient` from the `square` package.

```ts
import { SquareClient } from "square"

const client = new SquareClient({ token: "YOUR_TOKEN" })
```

The SDK also exports request and response types under a namespace.

```ts
import { Square } from "square"

const request: Square.CreatePaymentRequest = {
  sourceId: "ccof:GaJGNaZa8x4OgDJn4GB",
  idempotencyKey: "7b0f3ec5-086a-4871-8f13-3c81b3875218",
  amountMoney: {
    amount: BigInt("1000"),
    currency: "USD",
  },
}
```

### Commerce Kit Implication

Commerce Kit should keep its Better Auth-inspired inferred client, but Square validates that exported request and response types are still valuable. Inference is ergonomic for app code, while named exported types are useful for background jobs, service boundaries, tests, and integration packages.

## Resource Namespace Shape

Square uses generated domain namespaces with direct verb methods.

```ts
client.catalog.list(...)
client.catalog.search(...)
client.catalog.batchUpsert(...)

client.orders.create(...)
client.orders.calculate(...)
client.orders.update(...)
client.orders.pay(...)

client.payments.create(...)
client.payments.cancel(...)
client.payments.complete(...)

client.refunds.refundPayment(...)
client.customers.search(...)
client.checkout.paymentLinks.create(...)
client.webhooks.subscriptions.create(...)
```

Square mostly avoids chain builders. Deep domains become nested namespaces only where the API hierarchy is materially meaningful, such as `client.checkout.paymentLinks` and `client.webhooks.subscriptions`.

### Commerce Kit Implication

Commerce Kit should prefer domain namespaces over low-level REST path mirroring.

```ts
commerce.products.list(...)
commerce.carts.items.add(...)
commerce.carts.calculate(...)
commerce.carts.complete(...)
commerce.payments.authorize(...)
commerce.payments.capture(...)
commerce.refunds.create(...)
commerce.webhooks.verify(...)
```

Nested namespaces should be reserved for relationships users naturally think about, such as cart items, cart shipping methods, customer addresses, and webhook subscriptions.

## Request Options

Every Square method accepts request options after the typed request object. Options include API version override, headers, query params, max retries, timeout, abort signal, and raw response access.

```ts
await client.payments.create(
  {
    sourceId: "ccof:GaJGNaZa8x4OgDJn4GB",
    idempotencyKey: "7b0f3ec5-086a-4871-8f13-3c81b3875218",
    amountMoney: {
      amount: BigInt("1000"),
      currency: "USD",
    },
  },
  {
    version: "2024-05-04",
    timeoutInSeconds: 30,
    maxRetries: 0,
  },
)
```

Square also supports custom headers and query parameters.

```ts
await client.payments.create(..., {
  headers: {
    "X-Custom-Header": "custom value",
  },
  queryParams: {
    customQueryParamKey: "custom query param value",
  },
})
```

### Commerce Kit Implication

Commerce Kit should define a shared request options type for every client method.

```ts
type CommerceRequestOptions = {
  signal?: AbortSignal
  timeoutMs?: number
  retries?: number
  idempotencyKey?: string
  headers?: Record<string, string>
  raw?: boolean
}
```

Request options should be boring and consistent. Provider-specific adapters can consume them without leaking provider-specific SDKs into the Commerce Kit client.

## Money Representation

Square represents money with a currency code and integer minor-unit amount. The TypeScript SDK examples use `BigInt` for amounts.

```ts
amountMoney: {
  amount: BigInt("1599"),
  currency: "USD",
}
```

This appears across orders, payments, refunds, app fees, tips, discounts, and payment links.

### Commerce Kit Implication

Commerce Kit already intends to use integer minor-unit money. Square strengthens this decision, especially for financial operations where JavaScript `number` has precision risks.

Commerce Kit should decide whether public SDK input accepts `bigint`, stringified integer amounts, or a branded integer type. For JSON APIs, raw `bigint` cannot be serialized directly, so Commerce Kit should avoid exposing only `bigint` across network boundaries.

Recommended shape:

```ts
type Money = {
  amount: string
  currency: string
}
```

Server-side internal helpers can convert to `bigint` for calculations, but the HTTP/client contract should remain JSON-safe.

## Idempotency

Square requires or strongly encourages `idempotencyKey` on financial and mutating operations.

```ts
await client.orders.create({
  order: { ... },
  idempotencyKey: "8193148c-9586-11e6-99f9-28cfe92138cf",
})

await client.payments.create({
  sourceId: "ccof:GaJGNaZa8x4OgDJn4GB",
  idempotencyKey: "7b0f3ec5-086a-4871-8f13-3c81b3875218",
  amountMoney: {
    amount: BigInt("1000"),
    currency: "USD",
  },
})

await client.refunds.refundPayment({
  idempotencyKey: "9b7f2dcf-49da-4411-b23e-a2d6af21333a",
  amountMoney: {
    amount: BigInt("1000"),
    currency: "USD",
  },
  paymentId: "R2B3Z8WMVt3EAmzYWLZvz7Y69EbZY",
})
```

Square also exposes `payments.cancelByIdempotencyKey`, which is specifically designed for unknown network outcomes after a payment create request.

```ts
await client.payments.cancelByIdempotencyKey({
  idempotencyKey: "a7e36d40-d24b-11e8-b568-0800200c9a66",
})
```

### Commerce Kit Implication

Commerce Kit should make idempotency first-class for all irreversible or externally visible mutations.

High-priority idempotent methods:

- `carts.complete`
- `orders.create`
- `orders.cancel`
- `payments.authorize`
- `payments.capture`
- `payments.cancel`
- `refunds.create`
- `webhooks.process`

Commerce Kit should support automatic idempotency key generation for server-originated operations, while still allowing callers to provide deterministic keys for retries and background jobs.

## Pagination

Square wraps list responses in an async iterable page object.

```ts
const pageableResponse = await client.payments.list({
  beginTime: "begin_time",
  endTime: "end_time",
  cursor: "cursor",
  limit: 1,
})

for await (const item of pageableResponse) {
  console.log(item)
}
```

The same response can also be handled page by page.

```ts
let page = await client.payments.list({
  cursor: "cursor",
  limit: 1,
})

while (page.hasNextPage()) {
  page = page.getNextPage()
}

const response = page.response
```

### Commerce Kit Implication

Commerce Kit should expose a simple list response for app UI code and a richer iterator for admin or sync jobs.

Recommended UI-friendly default:

```ts
const { products, nextCursor } = await client.products.list({ limit: 20 })
```

Recommended advanced helper:

```ts
for await (const product of client.products.iterate({ limit: 100 })) {
  await indexProduct(product)
}
```

This keeps storefront usage simple while preserving Square-like ergonomics for batch processing.

## Error Handling

Square throws `SquareError` subclasses for non-2xx responses and exposes status, message, parsed body, and raw response.

```ts
import { SquareError } from "square"

try {
  await client.payments.create(...)
} catch (err) {
  if (err instanceof SquareError) {
    console.log(err.statusCode)
    console.log(err.message)
    console.log(err.body)
    console.log(err.rawResponse)
  }
}
```

### Commerce Kit Implication

Commerce Kit should use a single typed base error with stable domain codes.

```ts
class CommerceError extends Error {
  code: string
  status: number
  body?: unknown
  rawResponse?: Response
}
```

Important domain codes should not be provider-specific. Examples include `PAYMENT_DECLINED`, `PAYMENT_REQUIRES_ACTION`, `CART_NOT_FOUND`, `ORDER_VERSION_CONFLICT`, `IDEMPOTENCY_CONFLICT`, `WEBHOOK_SIGNATURE_INVALID`, and `ADAPTER_UNAVAILABLE`.

## Raw Response Access

Square supports `.withRawResponse()` on method calls.

```ts
const { data, rawResponse } = await client.payments.create(...).withRawResponse()

console.log(data)
console.log(rawResponse.headers["X-My-Header"])
```

List pages also expose the underlying response through `page.response`.

### Commerce Kit Implication

Commerce Kit should include a raw response escape hatch, especially for framework adapters, tests, observability, and provider debugging. This should not be the default API path, but it should exist without requiring users to bypass Commerce Kit entirely.

## Retries, Timeouts, Abort Signals, And Runtime Support

Square retries retryable requests by default with exponential backoff. Retryable responses include HTTP `408`, `429`, and `5xx`. The default retry limit is `2`.

Square also supports request-level timeouts, abort signals, a custom fetcher, and runtimes including Node.js 18+, Vercel, Cloudflare Workers, Deno, Bun, and React Native.

```ts
await client.payments.create(..., {
  maxRetries: 0,
  timeoutInSeconds: 30,
  abortSignal: controller.signal,
})
```

### Commerce Kit Implication

Commerce Kit should standardize retry and timeout behavior in core rather than making every adapter invent it independently. Adapters should be able to opt out for operations that are not safe to retry unless an idempotency key is present.

Commerce Kit should also keep runtime constraints explicit. Edge compatibility matters because ecommerce storefronts commonly run in Next.js, Remix, Cloudflare Workers, Vercel, and serverless functions.

## Webhooks

Square has API resources for webhook event types and subscriptions.

```ts
await client.webhooks.eventTypes.list({
  apiVersion: "api_version",
})

await client.webhooks.subscriptions.create({
  idempotencyKey: "63f84c6c-2200-4c99-846c-2670a1311fbf",
  subscription: {
    name: "Example Webhook Subscription",
    eventTypes: ["payment.created", "payment.updated"],
    notificationUrl: "https://example-webhook-url.com",
    apiVersion: "2021-12-15",
  },
})
```

The SDK also exposes webhook signature verification.

```ts
import { WebhooksHelper } from "square"

const isValid = WebhooksHelper.verifySignature({
  requestBody,
  signatureHeader: request.headers["x-square-hmacsha256-signature"],
  signatureKey: "YOUR_SIGNATURE_KEY",
  notificationUrl: "https://example.com/webhook",
})
```

### Commerce Kit Implication

Commerce Kit should provide a first-class webhook contract that handles raw body verification, event mapping, idempotency, and handler dispatch.

Recommended shape:

```ts
export const commerce = createCommerce({ ... })

export const POST = commerce.handler.webhook({
  provider: "stripe",
  onEvent: async (event) => {
    if (event.type === "payment.succeeded") {
      await commerce.orders.markPaid(event.data.orderId)
    }
  },
})
```

The webhook API should never require users to manually parse JSON before signature verification. Raw request access is mandatory.

## Catalog API

Square's catalog API is broad and lower-level than storefront product APIs. It exposes object-level batch operations, object search, item search, tax/modifier association updates, and API capability metadata.

```ts
await client.catalog.batchDelete({
  objectIds: ["W62UWFY35CWMYGVWK6TWJDNI", "AA27W3M2GGTF3H6AVPNB77CK"],
})

await client.catalog.batchGet({
  objectIds: ["W62UWFY35CWMYGVWK6TWJDNI", "AA27W3M2GGTF3H6AVPNB77CK"],
  includeRelatedObjects: true,
})

await client.catalog.batchUpsert({
  idempotencyKey: "789ff020-f723-43a9-b4b5-43b5dc1fa3dc",
  batches: [{ objects: [{ type: "ITEM", id: "id" }] }],
})

await client.catalog.searchItems({
  textFilter: "red",
  categoryIds: ["WINE_CATEGORY_ID"],
  stockLevels: ["OUT", "LOW"],
  enabledLocationIds: ["ATL_LOCATION_ID"],
  limit: 100,
})
```

### Commerce Kit Implication

Commerce Kit should not expose catalog internals as a generic object store to storefront users. It should expose product, variant, collection, category, price, inventory, and option concepts directly.

For admin/import APIs, Square's batch patterns are useful:

```ts
await commerce.admin.products.batchUpsert({
  idempotencyKey,
  products: [...],
})
```

Commerce Kit should separate storefront catalog APIs from admin catalog sync APIs.

## Orders API

Square orders are detailed commerce aggregates. They support line items, taxes, discounts, modifiers, references, locations, calculation previews, optimistic version updates, cloning, search, retrieval, and payment settlement.

```ts
await client.orders.create({
  order: {
    locationId: "057P5VYJ4A5X1",
    referenceId: "my-order-001",
    lineItems: [{
      name: "New York Strip Steak",
      quantity: "1",
      basePriceMoney: {
        amount: BigInt("1599"),
        currency: "USD",
      },
    }],
    taxes: [{
      uid: "state-sales-tax",
      name: "State Sales Tax",
      percentage: "9",
      scope: "ORDER",
    }],
  },
  idempotencyKey: "8193148c-9586-11e6-99f9-28cfe92138cf",
})
```

Calculation is separate from persistence.

```ts
await client.orders.calculate({
  order: {
    locationId: "D7AVYMEAPJ3A3",
    lineItems: [{
      name: "Item 1",
      quantity: "1",
      basePriceMoney: {
        amount: BigInt("500"),
        currency: "USD",
      },
    }],
  },
})
```

Updates require the latest order version.

```ts
await client.orders.update({
  orderId: "order_id",
  order: {
    locationId: "location_id",
    lineItems: [{ uid: "cookie_uid", quantity: "2" }],
    version: 1,
  },
  fieldsToClear: ["discounts"],
  idempotencyKey: "UNIQUE_STRING",
})
```

Paying an order is separate from creating payments.

```ts
await client.orders.pay({
  orderId: "order_id",
  idempotencyKey: "c043a359-7ad9-4136-82a9-c3f1d66dcbff",
  paymentIds: ["EnZdNAlWCmfh6Mt5FMNST1o7taB"],
})
```

### Commerce Kit Implication

Commerce Kit should keep order calculation, order creation, payment authorization, and order settlement as explicit concepts. A simple storefront checkout API can orchestrate them, but the underlying primitives should stay separate.

Recommended primitives:

```ts
await commerce.carts.calculate(cartId)
await commerce.orders.createFromCart(cartId, { idempotencyKey })
await commerce.payments.authorize(orderId, paymentInput, { idempotencyKey })
await commerce.orders.markPaid(orderId, paymentId, { idempotencyKey })
```

Square's `version` update pattern is also useful. Commerce Kit should use optimistic concurrency for order and cart mutations that can conflict.

## Payments API

Square payment operations separate authorization/creation, cancellation, capture/completion, updating approved payments, retrieval, listing, and canceling by idempotency key.

```ts
await client.payments.create({
  sourceId: "ccof:GaJGNaZa8x4OgDJn4GB",
  idempotencyKey: "7b0f3ec5-086a-4871-8f13-3c81b3875218",
  amountMoney: {
    amount: BigInt("1000"),
    currency: "USD",
  },
  autocomplete: true,
  customerId: "W92WH6P11H4Z77CTET0RNTGFW8",
  locationId: "L88917AVBK2S5",
  referenceId: "123456",
})

await client.payments.cancel({ paymentId: "payment_id" })
await client.payments.complete({ paymentId: "payment_id" })
```

Square's `autocomplete` flag controls immediate capture versus delayed capture. `complete` captures an approved delayed-capture payment.

### Commerce Kit Implication

Commerce Kit should avoid ambiguous payment method names like `create` for domain-level APIs. It should expose payment intent explicitly.

```ts
await commerce.payments.authorize({ ... })
await commerce.payments.capture(paymentId, { idempotencyKey })
await commerce.payments.cancel(paymentId, { idempotencyKey })
```

Provider adapters can map this to Square's `create`, `complete`, and `cancel`, Stripe's PaymentIntents, or other provider flows.

## Refunds API

Square exposes refunds separately from payments.

```ts
await client.refunds.refundPayment({
  idempotencyKey: "9b7f2dcf-49da-4411-b23e-a2d6af21333a",
  amountMoney: {
    amount: BigInt("1000"),
    currency: "USD",
  },
  paymentId: "R2B3Z8WMVt3EAmzYWLZvz7Y69EbZY",
  reason: "Example",
})

await client.refunds.get({ refundId: "refund_id" })
```

Refund listing is eventually consistent and paginated.

### Commerce Kit Implication

Commerce Kit should model refunds as first-class records, not just as payment status changes. Refunds need their own IDs, amounts, reasons, provider references, lifecycle states, webhooks, and idempotency keys.

## Checkout Payment Links

Square's checkout payment links API creates Square-hosted checkout pages.

```ts
await client.checkout.paymentLinks.create({
  idempotencyKey: "cd9e25dc-d9f2-4430-aedb-61605070e95f",
  quickPay: {
    name: "Auto Detailing",
    priceMoney: {
      amount: BigInt("10000"),
      currency: "USD",
    },
    locationId: "A9Y43N9ABXZBP",
  },
})
```

Payment links can be listed, retrieved, updated, and deleted. Updates are intentionally limited and cannot change fields such as `order_id`, `version`, URL, or timestamps.

### Commerce Kit Implication

Commerce Kit should support hosted-checkout-style payment sessions through payment adapters, but not force the whole checkout model to be hosted checkout.

Recommended abstraction:

```ts
await commerce.checkout.sessions.create({
  cartId,
  mode: "hosted",
  successUrl,
  cancelUrl,
})
```

The same order/cart primitives should still support embedded checkout and custom storefront flows.

## Customers API

Square's customers API includes list, create, batch create, bulk delete, bulk retrieve, bulk update, search, get, update, delete, groups, segments, cards, and custom attributes.

Create requires at least one identity field such as given name, family name, company name, email address, or phone number.

```ts
await client.customers.create({
  givenName: "Amelia",
  familyName: "Earhart",
  emailAddress: "Amelia.Earhart@example.com",
  phoneNumber: "+1-212-555-4240",
  referenceId: "YOUR_REFERENCE_ID",
})
```

Search supports structured filters.

```ts
await client.customers.search({
  limit: BigInt("2"),
  query: {
    filter: {
      emailAddress: {
        fuzzy: "example.com",
      },
    },
    sort: {
      field: "CREATED_AT",
      order: "ASC",
    },
  },
})
```

### Commerce Kit Implication

Commerce Kit's core should remain auth-agnostic, but customer records are still commerce domain objects. The API should distinguish customer profile management from authentication.

Recommended boundary:

```ts
await commerce.customers.create({
  email,
  name,
  externalUserId,
})

await commerce.customers.linkAuthUser(customerId, authUserId)
```

Auth providers can integrate through plugins without becoming part of core.

## API Versioning

Square pins the SDK to the latest API version by default but allows request-level version overrides.

```ts
await client.payments.create(..., {
  version: "2024-05-04",
})
```

Webhook subscriptions also carry an `apiVersion`.

### Commerce Kit Implication

Commerce Kit's own APIs should not need date-based external API versions early, but provider adapters should record the provider API version used for payments, refunds, and webhook events. This matters for debugging old orders and replaying old webhooks.

Recommended persisted metadata:

```ts
provider: "square"
providerApiVersion: "2024-05-04"
providerRequestId: "..."
providerObjectId: "..."
```

## Logging And Observability

Square supports configurable logging with log levels and custom logger implementations.

```ts
import { SquareClient, logging } from "square"

const client = new SquareClient({
  logging: {
    level: logging.LogLevel.Debug,
    logger: new logging.ConsoleLogger(),
    silent: false,
  },
})
```

### Commerce Kit Implication

Commerce Kit should expose logger hooks in core config.

```ts
const commerce = createCommerce({
  logger: {
    debug(message, context) {},
    info(message, context) {},
    warn(message, context) {},
    error(message, context) {},
  },
})
```

Payment, webhook, order state transition, adapter retry, and idempotency events should be observable without requiring users to patch internals.

## Strengths

- Consistent generated namespace and method shape across a large API surface.
- Strong financial API safety through idempotency keys and explicit cancellation/capture flows.
- Integer minor-unit money representation is consistent across domains.
- Async iterable pagination is excellent for batch processing and sync jobs.
- Request-level options cover real production needs without custom wrappers.
- Typed errors expose status, parsed body, and raw response.
- Webhook signature verification is available in the SDK.
- Subpackage exports allow tree-shaking and narrower imports.
- Runtime compatibility is explicit and broad.

## Weaknesses

- The API is provider-platform-shaped rather than commerce-engine-shaped, so some domain names are less ideal for Commerce Kit.
- Generated request objects can be verbose for storefront app developers.
- `BigInt` examples are safe for calculations but awkward for JSON and browser/server boundaries.
- Many methods are low-level and require the developer to understand Square-specific lifecycle details.
- `create` can be ambiguous for payments because it can mean authorize, capture, or external-record depending on request fields.

## Commerce Kit Priority Takeaways

- Make idempotency first-class for all payment, refund, order, cart completion, and webhook processing mutations.
- Keep money as integer minor units, but use JSON-safe API shapes rather than requiring `bigint` over the wire.
- Add shared request options for timeout, retries, abort signal, headers, idempotency key, and raw response access.
- Provide typed domain errors with stable Commerce Kit codes and access to provider/raw details.
- Use cursor pagination with a simple default response and optional async iterator helpers.
- Separate storefront-friendly APIs from admin/batch/sync APIs.
- Keep payment operations explicit: authorize, capture, cancel, refund, and settle order.
- Model refunds as first-class records.
- Provide raw-body-safe webhook handlers with signature verification and idempotent processing.
- Persist provider metadata such as provider object ID, request ID, and API version for financial traceability.
