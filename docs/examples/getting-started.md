# Getting started example

> This guide shows a **proposed v1 usage shape** for Commerce Kit. It is intentionally illustrative and does **not** imply that every function signature, package name, or framework helper below is already final or copy-paste runnable.

## What this example demonstrates

This example shows how a small starter app could compose Commerce Kit around one store backend:

- a required database adapter
- a required payment adapter
- optional fulfillment adapters
- one plugin

It focuses on the overall app shape and the responsibilities of each slot, without repeating the deeper architecture reference docs.

The default mental model here is a single store. Marketplace concepts are not part of the base config; they arrive later through a marketplace plugin if the business expands in that direction.

## Proposed project structure

```text
my-store/
├─ app/
│  └─ api/
│     ├─ commerce/
│     │  └─ [[...commerce]]/route.ts
│     └─ webhooks/
│        ├─ payment/[adapterId]/route.ts
│        └─ fulfillment/[adapterId]/route.ts
├─ lib/
│  ├─ commerce.ts
│  ├─ db.ts
│  └─ auth.ts
├─ plugins/
│  └─ internal-audits.ts
└─ env.ts
```

- `lib/commerce.ts` centralizes `createCommerce(...)`
- `app/api/...` mounts framework routes and webhook handlers
- `plugins/` holds app- or package-level Commerce Kit plugins

## Proposed `lib/commerce.ts`

```ts
import { createCommerce, drizzleAdapter } from "commerce-kit"
import { moyasarPayments } from "@commerce-kit/moyasar"
import { aramexFulfillment } from "@commerce-kit/aramex"
import { marsoolFulfillment } from "@commerce-kit/marsool"
import { couponsPlugin } from "@commerce-kit/coupons"
import { db } from "./db"
import { env } from "../env"

export const commerce = createCommerce({
  database: drizzleAdapter({
    db,
    provider: "postgres",
  }),

  payments: [
    moyasarPayments({
      adapterId: "moyasar-main",
      apiKey: env.MOYASAR_API_KEY,
      webhookSecret: env.MOYASAR_WEBHOOK_SECRET,
    }),
  ],

  fulfillment: [
    aramexFulfillment({
      accountNumber: env.ARAMEX_ACCOUNT_NUMBER,
      accountPin: env.ARAMEX_ACCOUNT_PIN,
    }),
    marsoolFulfillment({
      apiKey: env.MARSOOL_API_KEY,
    }),
  ],

  plugins: [
    couponsPlugin(),
  ],
})
```

Notes about the shape above:

- `database` is the required persistence boundary for all core reads and writes.
- `payments` is required and may contain more than one provider over time.
- `fulfillment` is optional and may include carrier shipping, local delivery, pickup, or digital fulfillment adapters behind one public API.
- `plugins` extend behavior without changing core adapter ownership. In this starter example, the plugin is still core-store-oriented rather than marketplace-oriented.
- `resolveContext` is intentionally not configured inline here; the framework layer below supplies it from app-owned auth/session code.
- Package choices use the main example packages described in the architecture docs where available.

## Proposed developer-facing API shape

The primary developer mental model should be a domain-first client surface:

```ts
const product = await commerce.products.create({
  title: "Classic linen shirt",
  slug: "classic-linen-shirt",
  variants: [
    {
      sku: "LINEN-S",
      price: { amount: 12999, currency: "USD" },
    },
  ],
})

commerce.cart.add({
  variantId: product.variants[0].id,
  quantity: 1,
})

const methods = await commerce.fulfillment.methods.list({
  cart: commerce.cart.get(),
  address: {
    country: "US",
    city: "New York",
    postalCode: "10001",
  },
})

const checkout = await commerce.orders.checkout({
  cart: commerce.cart.get(),
  fulfillment: {
    methodId: methods[0].id,
  },
  payment: {
    providerId: "moyasar",
  },
})

const products = await commerce.products.list({
  where: {
    status: "active",
    categoryId: { in: ["shirts", "summer"] },
  },
  pricing: {
    currency: "USD",
  },
  fields: ["id", "title", "slug", "price"],
  expand: ["variants", "media", "categories"],
  orderBy: [
    { createdAt: "desc" },
    { title: "asc" },
  ],
  limit: 20,
  cursor: "next_cursor",
})

await commerce.products.update({
  id: product.id,
  data: {
    title: "Classic linen shirt v2",
  },
})

await commerce.orders.get({ id: checkout.orderId })
```

Important behavior of this shape:

- the public API is `commerce.domain.method(...)`
- mutable namespaces expose only the CRUD-like methods they actually support
- immutable or workflow-driven namespaces omit unsupported methods rather than exposing them and failing later
- domain workflows such as `orders.checkout(...)`, `orders.cancel(...)`, and `orders.refund(...)` stay explicit instead of being forced into generic CRUD names

If a namespace needs search semantics beyond structured filtering, it may also expose a domain-specific `search` method:

```ts
const searchResults = await commerce.products.search({
  query: "linen shirt",
  where: {
    status: "active",
    inventory: { gt: 0 },
  },
  orderBy: [{ relevance: "desc" }],
  limit: 12,
})
```

`list` is still the standard collection reader. `search` should exist only when a namespace needs semantics beyond normal filtering and sorting.

### Product versioning

Products maintain a full version history to preserve historical pricing and details:

```ts
const product = await commerce.products.update({
  id: product.id,
  data: {
    price: { amount: 2999, currency: "USD" },
  },
})

const history = await commerce.products.versions({
  id: product.id,
  orderBy: [{ version: "desc" }],
})

const oldVersion = await commerce.products.getVersion({
  id: product.id,
  version: 1,
})
```

Each version records the product state at that point in time, useful for:
- Auditing price changes
- Historical order totals verification
- Tracking category and description changes

## Adding marketplace later

If the store later needs vendors, split orders, or payout workflows, that should be introduced by installing a marketplace plugin rather than by changing the base example's mental model.

Illustratively, that later expansion could look like:

```ts
import { marketplacePlugin } from "@commerce-kit/marketplace"

export const commerce = createCommerce({
  database: drizzleAdapter({ db, provider: "postgres" }),
  payments: [moyasarPayments({ adapterId: "moyasar-main" })],
  plugins: [
    couponsPlugin(),
    marketplacePlugin({
      payouts: true,
      splitOrdersByVendor: true,
    }),
  ],
})
```

At that point, vendor entities, vendor-owned product relationships, marketplace APIs/routes, and payout orchestration would be plugin-owned surfaces rather than assumptions baked into core.

## Example framework mounting

For a Next.js App Router style setup, a future `@commerce-kit/nextjs` package would likely provide the framework adapter. The example below shows the intended pattern: one shared route handler for normal HTTP verbs plus separate webhook handling.

Your app still owns authentication and session lookup. For example, `lib/auth.ts` might call your auth system directly. If useful, it could use `@commerce-kit/better-auth` as a thin bridge, but your app still decides how `resolveContext` is produced.

```ts
// lib/auth.ts
export async function resolveSession(req: Request) {
  // illustrative only: app-owned auth/session lookup
  return {
    customerId: null,
    locale: "en-US",
  }
}
```

Then the catch-all API route could pass requests through a shared Commerce Kit handler, using placeholder helper names:

```ts
// app/api/commerce/[[...commerce]]/route.ts
import { commerce } from "@/lib/commerce"
import { resolveSession } from "@/lib/auth"
import { createRouteHandler } from "@commerce-kit/nextjs"

const handler = createRouteHandler(commerce, {
  resolveContext: async (req) => resolveSession(req),
})

export { handler as GET, handler as POST, handler as PATCH, handler as DELETE }
```

In this example, the main Commerce Kit HTTP surface is mounted under `/api/commerce`, while provider webhooks stay under `/api/webhooks/...`.

Webhook routes would stay separate so each adapter surface can preserve and verify the raw body correctly:

```ts
// app/api/webhooks/payment/[adapterId]/route.ts
import { commerce } from "@/lib/commerce"
import { createWebhookHandler } from "@commerce-kit/nextjs"

export async function POST(
  req: Request,
  { params }: { params: { adapterId: string } },
) {
  return createWebhookHandler(commerce, {
    type: "payment",
    adapterId: params.adapterId,
  })(req)
}
```

The exact helper names above are still illustrative. The important part is the ownership split: your app supplies auth/context resolution, while the framework adapter maps HTTP requests into Commerce Kit and keeps webhook handling separate.

## Example request flow

One realistic flow for this starter app could look like this:

1. A storefront calls `POST /api/commerce/orders/checkout` with cart, fulfillment, and payment details.
2. Commerce Kit validates input and resolves request context through `resolveContext`.
3. Core order logic reads and writes through the configured `database` adapter.
4. The selected `payment` adapter authorizes or captures payment.
5. If the order needs fulfillment, Commerce Kit resolves the selected configured fulfillment method.
6. The active fulfillment adapter can create carrier shipping, local delivery, pickup, or digital fulfillment work as needed.
7. Installed `plugins` can add hooks, states, schema, or extra typed APIs.
8. Provider webhooks later re-enter through dedicated `/api/webhooks/...` endpoints and update Commerce Kit state after signature verification.

## What each adapter/plugin slot is for

| Slot | Purpose |
|---|---|
| `database` | Required storage contract for Commerce Kit data and transactions. |
| `payments` | Required payment provider integrations for authorize, capture, refund, and webhook verification. |
| `fulfillment` | Optional fulfillment integrations for carrier shipping, local delivery, pickup, digital delivery, labels, tracking, and configured method pricing. |
| `plugins` | Feature or domain extensions that can add hooks, schema, state transitions, server APIs, client APIs, and verified webhook handlers. Marketplace is one example of a plugin-owned domain expansion. |

## Optional adapter note

Fulfillment is optional in the proposed v1 shape. If your app only needs payments, or handles fulfillment outside Commerce Kit, you can leave the adapter array out.

In practice, that just means Commerce Kit would not add fulfillment-specific behavior for that app.

So a minimal app might start with only:

- `database`
- `payments`
- zero or more `plugins`

...and add `fulfillment` later when the product actually needs it.
