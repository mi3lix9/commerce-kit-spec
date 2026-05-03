# Medusa Store JS SDK Research

## Purpose

This document captures detailed API findings from the Medusa JS SDK Store reference at `https://docs.medusajs.com/resources/references/js-sdk/store/`. It focuses on the storefront-facing `sdk.store` API and the design lessons relevant to Commerce Kit.

The Medusa Store SDK is a strong reference for a typed storefront client organized around commerce domains. It is less Better Auth-like than Commerce Kit's intended server architecture because Medusa is a platform with a predefined backend, but its Store SDK contains useful API details for catalog, cart, checkout, payment, fulfillment, localization, regions, and customer account flows.

## Store SDK Surface

The Store SDK exposes a `sdk.store` namespace with the following resources:

- `sdk.store.cart`
- `sdk.store.category`
- `sdk.store.collection`
- `sdk.store.customer`
- `sdk.store.fulfillment`
- `sdk.store.locale`
- `sdk.store.order`
- `sdk.store.payment`
- `sdk.store.product`
- `sdk.store.region`

The API is method-based rather than nested-resource-builder-based. For example, cart line item operations are exposed as `createLineItem`, `updateLineItem`, and `deleteLineItem` under `sdk.store.cart`, not as `sdk.store.cart.items.create`.

```ts
sdk.store.product.list(...)
sdk.store.product.retrieve(...)

sdk.store.cart.create(...)
sdk.store.cart.update(...)
sdk.store.cart.retrieve(...)
sdk.store.cart.createLineItem(...)
sdk.store.cart.addShippingMethod(...)
sdk.store.cart.complete(...)

sdk.store.payment.listPaymentProviders(...)
sdk.store.payment.initiatePaymentSession(...)

sdk.store.fulfillment.listCartOptions(...)
sdk.store.fulfillment.calculate(...)

sdk.store.customer.retrieve(...)
sdk.store.customer.createAddress(...)
sdk.store.customer.listAddress(...)
```

## Shared API Conventions

### Response Envelopes

Most methods return an object envelope named after the returned resource.

```ts
sdk.store.product.retrieve("prod_123")
  .then(({ product }) => {
    console.log(product)
  })

sdk.store.cart.retrieve("cart_123")
  .then(({ cart }) => {
    console.log(cart)
  })
```

List responses usually return resources plus pagination metadata.

```ts
sdk.store.product.list()
  .then(({ products, count, offset, limit }) => {
    console.log(products)
  })
```

Some list responses also include `estimate_count`, which may be inaccurate because it comes from the PostgreSQL query planner.

### Pagination

Medusa uses offset pagination in the Store SDK.

```ts
sdk.store.product.list({
  limit: 10,
  offset: 10,
})
```

This is simple and familiar, but Commerce Kit's draft docs currently prefer cursor pagination. Cursor pagination is better for long-lived storefront pagination and concurrent writes, while offset pagination is easier to explain and debug.

### Field Selection and Relation Loading

Medusa uses a string-based `fields` parameter for both selecting fields and expanding relations.

```ts
sdk.store.product.list({
  fields: "id,*collection",
})

sdk.store.cart.retrieve("cart_123", {
  fields: "id,*items",
})
```

Observed conventions:

- Fields and relations are comma-separated strings.
- `*relation` is used to retrieve a relation.
- Deep relation/totals retrieval can require expanded relation patterns such as `items.*` or `shipping_methods.*`.
- The same `fields` mechanism appears across products, carts, orders, categories, collections, regions, payment collections, and addresses.

Commerce Kit should borrow the idea of one consistent selection mechanism, but should prefer a typed array form for public SDK ergonomics.

```ts
await client.products.get("prod_123", {
  fields: ["id", "title", "slug"],
  expand: ["variants", "collection"],
})
```

### Query Operators

Medusa list params support filter objects with logical operators and scalar operators.

Examples from product, category, collection, region, order, and address filters include:

- `$and`
- `$or`
- `$eq`
- `$ne`
- `$in`
- `$nin`
- `$not`
- `$gt`
- `$gte`
- `$lt`
- `$lte`
- `$like`
- `$re`
- `$ilike`
- `$fulltext`
- `$overlap`
- `$contains`
- `$contained`
- `$exists`

Product list examples include direct filters such as `status`, `sales_channel_id`, `title`, `handle`, `external_id`, `id`, `is_giftcard`, `tag_id`, `type_id`, `category_id`, `collection_id`, date filters, and variant filters.

```ts
sdk.store.product.list({
  q: "shirt",
  category_id: "pcat_123",
  region_id: "reg_123",
  country_code: "us",
})
```

Commerce Kit should take the typed operator model seriously, but expose it under a clearer `where` property rather than mixing filter keys, pagination keys, field selection, and localization keys in a single object.

```ts
await client.products.list({
  where: {
    title: { contains: "shirt" },
    categoryId: "cat_123",
    createdAt: { gte: "2026-01-01" },
  },
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
})
```

### Headers and Next.js Cache Tags

Most methods accept a final `headers` argument. The documentation repeatedly calls out `tags` for Next.js cache tagging.

```ts
sdk.store.product.list(
  { limit: 10 },
  { tags: ["products"] },
)
```

Commerce Kit should support framework-specific request metadata without hard-coding Next.js into core.

```ts
await client.products.list(
  { limit: 10 },
  { next: { tags: ["products"] } },
)
```

## Product API

### Methods

`sdk.store.product` exposes:

- `list(query?, headers?)`
- `retrieve(id, query?, headers?)`

### List Products

```ts
sdk.store.product.list()
  .then(({ products, count, offset, limit }) => {
    console.log(products)
  })
```

Pagination:

```ts
sdk.store.product.list({
  limit: 10,
  offset: 10,
})
```

Field selection:

```ts
sdk.store.product.list({
  fields: "id,*collection",
})
```

Important query parameters:

- `fields`
- `limit`
- `offset`
- `order`
- `q`
- `status`
- `sales_channel_id`
- `title`
- `handle`
- `external_id`
- `id`
- `is_giftcard`
- `tag_id`
- `type_id`
- `category_id`
- `categories`
- `collection_id`
- `created_at`
- `updated_at`
- `deleted_at`
- `region_id`
- `country_code`
- `province`
- `cart_id`
- `variants`
- `locale`

Pricing and tax context can come from `region_id`, `country_code`, `province`, or `cart_id`. If `cart_id` is provided, Medusa uses the cart's region and shipping address context.

### Retrieve Product

```ts
sdk.store.product.retrieve("prod_123")
  .then(({ product }) => {
    console.log(product)
  })
```

With selected fields and relations:

```ts
sdk.store.product.retrieve("prod_123", {
  fields: "id,*collection",
})
```

Retrieve accepts pricing/tax context too:

- `region_id`
- `country_code`
- `province`
- `cart_id`
- `locale`

### Product Entity Shape

The product shape includes fields such as:

- `id`
- `title`
- `handle`
- `subtitle`
- `description`
- `is_giftcard`
- `status`
- `thumbnail`
- dimensions and weight
- customs fields such as origin country, HS code, MID code, material
- `collection_id`
- `type_id`
- `discountable`
- `external_id`
- timestamps
- `variants`
- `options`
- `images`
- `metadata`
- `collection`
- `categories`
- `type`
- `tags`

### Commerce Kit Lessons

- Product reads need pricing context. Commerce Kit should support market/cart/customer context for catalog prices.
- `cartId` as product price context is valuable because customer-specific shipping/tax region may already live on the cart.
- Field/relation selection matters for storefront performance.
- Product list filtering needs both domain filters and generic date/operator filters.

## Category API

### Methods

`sdk.store.category` exposes:

- `list(query?, headers?)`
- `retrieve(id, query?, headers?)`

### List Categories

```ts
sdk.store.category.list()
  .then(({ product_categories, count, offset, limit }) => {
    console.log(product_categories)
  })
```

Pagination and fields:

```ts
sdk.store.category.list({
  limit: 10,
  offset: 10,
  fields: "id,*parent_category",
})
```

Important filters:

- `q`
- `id`
- `name`
- `description`
- `parent_category_id`
- `handle`
- `external_id`
- `is_active`
- `is_internal`
- `include_descendants_tree`
- `include_ancestors_tree`
- `created_at`
- `updated_at`
- `deleted_at`

### Retrieve Category

```ts
sdk.store.category.retrieve("pcat_123")
  .then(({ product_category }) => {
    console.log(product_category)
  })
```

Tree controls:

```ts
sdk.store.category.retrieve("pcat_123", {
  include_ancestors_tree: true,
  include_descendants_tree: true,
})
```

### Category Entity Shape

The category shape includes:

- `id`
- `name`
- `description`
- `handle`
- `rank`
- `external_id`
- `parent_category_id`
- timestamps
- `parent_category`
- `category_children`
- `metadata`
- `products`

### Commerce Kit Lessons

- Category tree loading should be explicit. Commerce Kit should support `includeAncestors` and `includeDescendants`, or typed expand paths such as `ancestors` and `children`.
- `handle` or `slug` based retrieval is important for storefront URLs. Medusa retrieve uses ID; Spree supports permalink/slug patterns. Commerce Kit should allow `getBySlug` or `get(slugOrId)` if slug lookup is first-class.

## Collection API

### Methods

`sdk.store.collection` exposes:

- `list(query?, headers?)`
- `retrieve(id, query?, headers?)`

### List Collections

```ts
sdk.store.collection.list()
  .then(({ collections, count, limit, offset }) => {
    console.log(collections)
  })
```

Useful filters:

- `q`
- `id`
- `handle`
- `title`
- `external_id`
- `created_at`
- `updated_at`
- `fields`
- `limit`
- `offset`
- `order`

### Retrieve Collection

```ts
sdk.store.collection.retrieve("pcol_123", {
  fields: "id,handle",
})
```

### Collection Entity Shape

The collection shape includes:

- `id`
- `title`
- `handle`
- timestamps
- `metadata`
- `external_id`
- `products`

### Commerce Kit Lessons

- Collections are distinct from categories. Categories model taxonomy; collections model merchandising groupings.
- Commerce Kit should decide whether `collections` are core v1 or plugin-owned merchandising. If included, use a separate namespace from `categories`.

## Cart API

### Methods

`sdk.store.cart` exposes:

- `create(body?, query?, headers?)`
- `update(id, body, query?, headers?)`
- `retrieve(id, query?, headers?)`
- `createLineItem(cartId, body, query?, headers?)`
- `updateLineItem(cartId, lineItemId, body, query?, headers?)`
- `deleteLineItem(cartId, lineItemId, query?, headers?)`
- `addShippingMethod(cartId, body, query?, headers?)`
- `addPromotions(cartId, body, query?, headers?)`
- `removePromotions(cartId, body, query?, headers?)`
- `complete(cartId, query?, headers?)`
- `transferCart(cartId, body?, query?, headers?)`

### Create Cart

```ts
sdk.store.cart.create({
  region_id: "reg_123",
})
```

Create cart can accept a substantial initial payload:

- `region_id`
- `shipping_address`
- `billing_address`
- `email`
- `currency_code`
- `items`
- `sales_channel_id`
- `promo_codes`
- `metadata`
- `locale`

This allows one-shot cart creation with region, customer email, addresses, items, promo codes, and metadata.

### Update Cart

```ts
sdk.store.cart.update("cart_123", {
  region_id: "reg_123",
})
```

Update can change cart region, addresses, email, sales channel, metadata, promo codes, and locale.

### Retrieve Cart

```ts
sdk.store.cart.retrieve("cart_123", {
  fields: "id,*items",
})
```

### Line Items

```ts
sdk.store.cart.createLineItem("cart_123", {
  variant_id: "variant_123",
  quantity: 1,
})

sdk.store.cart.updateLineItem("cart_123", "li_123", {
  quantity: 2,
})

sdk.store.cart.deleteLineItem("cart_123", "li_123")
```

Delete returns a deletion envelope with `id`, `object`, `deleted`, and `parent`.

```ts
sdk.store.cart.deleteLineItem("cart_123", "li_123")
  .then(({ deleted, parent: cart }) => {
    console.log(deleted, cart)
  })
```

### Shipping Method Selection

```ts
sdk.store.cart.addShippingMethod("cart_123", {
  option_id: "so_123",
  data: {
    // fulfillment provider data
  },
})
```

This is a useful pattern: the storefront chooses a shipping option and can pass provider-specific data through `data`.

### Promotions

```ts
sdk.store.cart.addPromotions("cart_123", {
  promo_codes: ["20OFF"],
})

sdk.store.cart.removePromotions("cart_123", {
  promo_codes: ["20OFF"],
})
```

### Complete Cart

```ts
sdk.store.cart.complete("cart_123")
  .then(({ order }) => {
    console.log(order)
  })
```

Cart completion produces an order. This reinforces the domain distinction between mutable cart and immutable placed order.

### Transfer Cart

Medusa supports cart transfer behavior. This is useful when attaching an anonymous cart to a logged-in customer.

### Cart Totals

The cart response includes a rich set of totals:

- `original_item_total`
- `original_item_subtotal`
- `original_item_tax_total`
- `item_total`
- `item_subtotal`
- `item_tax_total`
- `original_total`
- `original_subtotal`
- `original_tax_total`
- `total`
- `subtotal`
- `tax_total`
- `discount_total`
- `discount_tax_total`
- `gift_card_total`
- `gift_card_tax_total`
- `shipping_total`
- `shipping_subtotal`
- `shipping_tax_total`
- `original_shipping_total`
- `original_shipping_subtotal`
- `original_shipping_tax_total`

The response also includes:

- `promotions`
- `region_id`
- `customer_id`
- `sales_channel_id`
- `email`
- `metadata`
- timestamps
- `shipping_address`
- `billing_address`
- `items`
- `shipping_methods`
- `payment_collection`
- `region`

### Commerce Kit Lessons

- Cart should be a first-class aggregate with rich totals and selected shipping/payment state.
- Commerce Kit should use nested resources for cart item operations rather than Medusa's flattened method names. `client.carts.items.create(cartId, ...)` is clearer than `client.carts.createLineItem(cartId, ...)`.
- Cart creation should support one-shot creation with initial items, customer email, addresses, and promo codes.
- Shipping method selection should support provider-specific `data` while keeping it isolated.
- Promotion add/remove methods should be explicit workflow actions.
- Cart completion should return an order and should clearly mark the cart as no longer mutable.

## Fulfillment API

### Methods

`sdk.store.fulfillment` exposes:

- `listCartOptions(query, headers?)`
- `calculate(id, body, query?, headers?)`

### List Cart Shipping Options

```ts
sdk.store.fulfillment.listCartOptions({
  cart_id: "cart_123",
})
  .then(({ shipping_options }) => {
    console.log(shipping_options)
  })
```

Important query parameters:

- `cart_id`
- `fields`
- `limit`
- `offset`
- `order`
- `is_return`

Shipping option shape includes:

- `id`
- `name`
- `price_type`
- `service_zone_id`
- `shipping_profile_id`
- `provider_id`
- `data`
- `type`
- `provider`
- `amount`
- `prices`
- `calculated_price`
- `insufficient_inventory`
- `service_zone`

### Calculate Shipping Option

```ts
sdk.store.fulfillment.calculate("so_123", {
  cart_id: "cart_123",
  data: {
    // provider-specific calculation data
  },
})
  .then(({ shipping_option }) => {
    console.log(shipping_option)
  })
```

### Commerce Kit Lessons

- Shipping option listing and shipping option calculation are distinct operations.
- A shipping/delivery adapter should declare whether a rate is fixed or calculated.
- `insufficient_inventory` on shipping options is an important checkout signal because fulfillment availability is not only about price.
- Provider-specific fulfillment data should be supported but isolated under a typed or adapter-owned `data` field.

## Payment API

### Methods

`sdk.store.payment` exposes:

- `listPaymentProviders(query, headers?)`
- `initiatePaymentSession(cart, body, query?, headers?)`

### List Payment Providers

```ts
sdk.store.payment.listPaymentProviders({
  region_id: "reg_123",
})
  .then(({ payment_providers, count, offset, limit }) => {
    console.log(payment_providers)
  })
```

Payment providers are scoped by region.

### Initiate Payment Session

```ts
sdk.store.payment.initiatePaymentSession(
  cart,
  {
    provider_id: "pp_stripe_stripe",
    data: {
      // payment-provider-specific data
    },
  },
)
  .then(({ payment_collection }) => {
    console.log(payment_collection)
  })
```

The method accepts the full cart object, not just `cartId`. If the cart has no payment collection, Medusa creates one before initializing the payment session.

The payment collection shape includes:

- `id`
- `currency_code`
- `amount`
- `status`
- `payment_providers`
- `authorized_amount`
- `captured_amount`
- `refunded_amount`
- `completed_at`
- timestamps
- `metadata`
- `payments`
- `payment_sessions`

Payment session body includes:

- `provider_id`
- `data`

### Commerce Kit Lessons

- Payment providers should be scoped by region, market, or configured checkout context.
- Payment session initialization should not require the caller to manage internal payment collection setup.
- Commerce Kit should probably accept `cartId`, not a full cart object, for client ergonomics and stale-object avoidance.
- Provider-specific payment data should live under `data`, `externalData`, or an adapter-owned type.
- Payment collection is a useful internal aggregate, but public SDK naming may be simpler as `carts.paymentSessions.create(...)`.

## Order API

### Methods

`sdk.store.order` exposes:

- `list(query?, headers?)`
- `retrieve(id, query?, headers?)`
- `requestTransfer(id, body, query?, headers?)`
- `cancelTransfer(id, query?, headers?)`
- `acceptTransfer(id, body, query?, headers?)`
- `declineTransfer(id, body, query?, headers?)`

### List Orders

```ts
sdk.store.order.list({
  limit: 10,
  offset: 10,
  fields: "id,*items",
})
  .then(({ orders, count, offset, limit }) => {
    console.log(orders)
  })
```

Store order list requires customer authentication.

Important filters:

- `id`
- `status`
- `fields`
- `limit`
- `offset`
- `order`
- `with_deleted`
- `$and`
- `$or`

### Retrieve Order

```ts
sdk.store.order.retrieve("order_123", {
  fields: "id,*items",
})
```

### Order Transfer

Medusa includes order transfer workflows:

```ts
sdk.store.order.requestTransfer(
  "order_123",
  {
    description: "I want to transfer this order to my friend.",
    update_order_email: true,
  },
  {},
  { Authorization: `Bearer ${token}` },
)
```

```ts
sdk.store.order.acceptTransfer(
  "order_123",
  { token: "transfer_token" },
  {},
  { Authorization: `Bearer ${token}` },
)
```

There are also `cancelTransfer` and `declineTransfer` methods.

### Order Entity Shape

The order shape includes:

- IDs for region, customer, and sales channel
- `email`
- `currency_code`
- `status`
- `payment_status`
- `fulfillment_status`
- `summary`
- timestamps
- item, shipping, tax, discount, gift card, and credit-line totals
- `items`
- `shipping_methods`
- `display_id`
- `custom_display_id`
- `transactions`
- `metadata`
- addresses
- `payment_collections`
- `fulfillments`
- `customer`

### Commerce Kit Lessons

- Storefront order reads should be separate from admin/server order mutations.
- Order transfer is a valuable workflow but should not be core v1 unless Commerce Kit explicitly supports guest-to-customer order reassignment.
- Order totals need to preserve original totals, adjusted totals, tax totals, shipping totals, discount totals, and credit lines.

## Customer API

### Methods

`sdk.store.customer` exposes:

- `create(body, query?, headers?)`
- `update(body, query?, headers?)`
- `retrieve(query?, headers?)`
- `createAddress(body, query?, headers?)`
- `updateAddress(addressId, body, query?, headers?)`
- `listAddress(query?, headers?)`
- `retrieveAddress(addressId, query?, headers?)`
- `deleteAddress(addressId, headers?)`

### Register Customer

Medusa customer creation is tied to its auth module. The docs require using `sdk.auth.register("customer", "emailpass", ...)` first to get a registration token, then passing that token in the `Authorization` header.

```ts
const token = await sdk.auth.register("customer", "emailpass", {
  email: "customer@gmail.com",
  password: "supersecret",
})

sdk.store.customer.create(
  { email: "customer@gmail.com" },
  {},
  { Authorization: `Bearer ${token}` },
)
```

### Retrieve and Update Current Customer

```ts
sdk.store.customer.retrieve()
  .then(({ customer }) => {
    console.log(customer)
  })

sdk.store.customer.update({
  first_name: "John",
})
```

These operations require the customer to be logged in.

### Address Management

```ts
sdk.store.customer.createAddress({
  country_code: "us",
  is_default_shipping: true,
  is_default_billing: true,
})

sdk.store.customer.updateAddress("caddr_123", {
  country_code: "us",
})

sdk.store.customer.listAddress({
  limit: 10,
  offset: 10,
  fields: "id,country_code",
})

sdk.store.customer.retrieveAddress("caddr_123", {
  fields: "id,country_code",
})

sdk.store.customer.deleteAddress("caddr_123")
```

Address inputs include first name, last name, phone, company, address lines, city, country code, province, postal code, metadata, address name, default shipping flag, and default billing flag.

### Commerce Kit Lessons

- Commerce Kit should remain auth-agnostic, unlike Medusa's auth-integrated storefront customer creation.
- A current-customer namespace is useful, but it should be backed by `resolveContext` rather than built-in auth.
- Address methods should probably be nested: `client.customer.addresses.create(...)` instead of `client.customer.createAddress(...)`.
- Default billing and default shipping flags should be explicit and transactional.

## Region API

### Methods

`sdk.store.region` exposes:

- `list(query?, headers?)`
- `retrieve(id, query?, headers?)`

### List Regions

```ts
sdk.store.region.list({
  limit: 10,
  offset: 10,
  fields: "id,*countries",
})
  .then(({ regions, count, limit, offset }) => {
    console.log(regions)
  })
```

Important filters:

- `q`
- `id`
- `name`
- `currency_code`
- `created_at`
- `updated_at`
- `fields`
- `limit`
- `offset`
- `order`

### Retrieve Region

```ts
sdk.store.region.retrieve("reg_123", {
  fields: "id,*countries",
})
```

Region shape includes:

- `id`
- `name`
- `currency_code`
- `automatic_taxes`
- `countries`
- `payment_providers`
- `metadata`
- timestamps

### Commerce Kit Lessons

- Medusa uses regions where Spree uses markets. Commerce Kit should choose one concept intentionally.
- `markets` is likely the better public word for a modern SDK because it can cover country, currency, locale, tax inclusivity, and sales configuration.
- Region/market context should influence product pricing, tax calculation, payment providers, shipping options, and cart creation.

## Locale API

### Methods

`sdk.store.locale` exposes:

- `list(headers?)`

### List Locales

```ts
sdk.store.locale.list()
  .then(({ locales }) => {
    console.log(locales)
  })
```

Locale shape includes:

- `code`
- `name`

### Commerce Kit Lessons

- Locale should be treated as request context rather than a global singleton.
- A locale listing endpoint is useful for storefront language selectors.
- If Commerce Kit supports markets, supported locales likely belong on market records.

## Storefront Checkout Flow In Medusa

A typical Store SDK checkout flow can be represented as:

```ts
const { cart } = await sdk.store.cart.create({
  region_id: "reg_123",
  email: "customer@example.com",
})

await sdk.store.cart.createLineItem(cart.id, {
  variant_id: "variant_123",
  quantity: 1,
})

await sdk.store.cart.update(cart.id, {
  shipping_address: {
    first_name: "John",
    last_name: "Doe",
    address_1: "123 Main St",
    city: "Riyadh",
    country_code: "sa",
    postal_code: "12345",
  },
})

const { shipping_options } = await sdk.store.fulfillment.listCartOptions({
  cart_id: cart.id,
})

await sdk.store.cart.addShippingMethod(cart.id, {
  option_id: shipping_options[0].id,
})

const { payment_providers } = await sdk.store.payment.listPaymentProviders({
  region_id: cart.region_id,
})

await sdk.store.payment.initiatePaymentSession(cart, {
  provider_id: payment_providers[0].id,
  data: {},
})

const { order } = await sdk.store.cart.complete(cart.id)
```

This flow has good domain separation but a few DX issues:

- Payment session initiation takes the full cart object rather than `cartId`.
- Cart item and address methods are flat names rather than nested namespaces.
- Field selection is string-based.
- Payment provider selection is region-scoped, which is useful but requires the developer to understand regions early.

## Strengths

- Store namespaces cover the core storefront lifecycle: catalog, category, collection, cart, fulfillment, payment, order, customer, region, locale.
- Cart is a rich aggregate with addresses, items, shipping methods, payment collection, region, promotions, and totals.
- Checkout separates shipping option listing, shipping option calculation, shipping method selection, payment provider listing, payment session initiation, and cart completion.
- Product reads support pricing/tax context through region, country, province, and cart.
- Field selection is consistent across resources.
- Query operators are powerful and typed.
- Customer account and address flows are covered.
- Region and locale APIs acknowledge global storefront needs.
- Next.js cache tags are supported through request headers.

## Weaknesses

- API method naming is less ergonomic than nested resources for children like line items and addresses.
- `fields` is a string DSL, which is flexible but less discoverable than typed arrays.
- Offset pagination is simple but weaker than cursor pagination for large or changing collections.
- Query objects mix filters, pagination, sorting, field selection, and contextual inputs in one object.
- Payment session initiation requires a full cart object, which can be stale and awkward.
- Customer APIs are tied to Medusa auth conventions; Commerce Kit should stay auth-agnostic.
- Region terminology may be less flexible than market terminology for Commerce Kit's long-term scope.

## Commerce Kit API Recommendations

### Use Nested Namespaces For Child Resources

Prefer this:

```ts
await client.carts.items.create(cartId, {
  variantId: "var_123",
  quantity: 1,
})

await client.customer.addresses.create({
  countryCode: "sa",
})
```

Over this:

```ts
await client.carts.createLineItem(cartId, ...)
await client.customer.createAddress(...)
```

### Split Query Shape From Context

Medusa puts pricing context, locale, filters, and pagination in one query object. Commerce Kit should separate collection query from request context.

```ts
await client.products.list(
  {
    where: {
      categoryId: "cat_123",
      title: { contains: "shirt" },
    },
    orderBy: [{ createdAt: "desc" }],
    limit: 20,
    cursor: "next_cursor",
    fields: ["id", "title", "slug"],
    expand: ["variants", "media"],
  },
  {
    marketId: "mkt_123",
    locale: "ar-SA",
    countryCode: "SA",
    cartId: "cart_123",
  },
)
```

### Keep Payment Sessions First-Class

Medusa's payment collection and session model is useful, but Commerce Kit should expose it through cart-owned payment sessions.

```ts
await client.carts.paymentSessions.create(cartId, {
  paymentAdapterId: "stripe-main",
  data: {
    returnUrl: "https://example.com/checkout/complete",
  },
})
```

### Support Shipping Option Calculation

Commerce Kit's shipping adapter surface should distinguish listing eligible options from calculating provider-backed option prices.

```ts
await client.shipping.options.list({
  cartId: "cart_123",
})

await client.shipping.options.calculate("option_123", {
  cartId: "cart_123",
  data: {},
})
```

If shipping is dormant, this namespace should not exist.

### Use Markets Instead Of Regions Publicly

Medusa's `region` covers currency, countries, tax automation, and payment providers. Spree's `market` concept is more expressive for storefront business configuration. Commerce Kit should likely expose `markets` publicly and map lower-level region concepts internally if needed.

```ts
await client.markets.list()
await client.markets.resolve({ countryCode: "SA" })
await client.markets.countries.list("mkt_123")
```

### Keep Money As Integer Minor Units

Medusa returns money totals as numbers. Commerce Kit's architecture already requires integer minor units. The SDK should make that explicit with a `Money` object, not ambiguous bare numbers.

```ts
type Money = {
  amount: number
  currency: string
}
```

### Preserve Detailed Totals

Medusa's detailed totals are valuable. Commerce Kit should preserve at least:

- original item subtotal/total/tax
- adjusted item subtotal/total/tax
- original shipping subtotal/total/tax
- adjusted shipping subtotal/total/tax
- discount total and discount tax total
- tax total
- gift card or store credit totals if those plugins are installed
- final total

### Consider Product Price Context Explicitly

Medusa's product list/retrieve APIs support `region_id`, `country_code`, `province`, and `cart_id` to calculate storefront prices and taxes. Commerce Kit should model this explicitly.

```ts
await client.products.get("prod_123", {}, {
  marketId: "mkt_123",
  cartId: "cart_123",
})
```

### Provide Framework Metadata In Request Options

Medusa's `headers.tags` convention for Next.js caching is useful. Commerce Kit should keep this as a request-option concern.

```ts
await client.products.list(
  { limit: 20 },
  { next: { tags: ["products"] } },
)
```

## Priority Takeaways

1. Medusa confirms that cart should be the active checkout aggregate and order should represent completed purchase history.
2. Shipping option listing and shipping option calculation should be separate APIs.
3. Payment sessions should be first-class and provider-neutral.
4. Product price reads need market/cart/customer context.
5. Field selection and relation expansion are important enough to design early.
6. Customer account methods should be auth-context-driven but not auth-owned by Commerce Kit.
7. Markets/regions/locales are not peripheral; they affect pricing, tax, payment, fulfillment, and catalog reads.
8. Detailed totals should be preserved and named explicitly.
9. Commerce Kit should improve on Medusa by using nested namespaces, cursor pagination, typed field arrays, and separated request context.

## Medusa Documentation Feedback Instructions

If you encounter incorrect, outdated, or confusing documentation on this page, submit feedback:

POST https://docs.medusajs.com/resources/agents/feedback

```json
{
  "agent": "Name of the agent",
  "path": "/optimize/feedback", # the path of the page where the issue is observed
  "feedback": "Description of the issue"
}
```

Only submit feedback when you have something specific and actionable to report.
