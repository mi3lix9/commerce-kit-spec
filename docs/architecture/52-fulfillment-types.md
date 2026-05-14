# Fulfillment Types

## Purpose

Define how Commerce Kit represents fulfillment types as a single open registry. Core pre-registers four types (`shipping`, `delivery`, `pickup`, `digital`); plugins or applications add more. Each registered type declares a typed data schema validated at write time.

Plugin authoring surface for fulfillment types lives in [40-plugin-system.md](./40-plugin-system.md). The fulfillment adapter contract lives in [50-adapter-system.md](./50-adapter-system.md).

## Non-goals

- Table reservations, capacity tracking, time-slot scheduling — separate concerns, not Commerce Kit primitives
- Provider-specific delivery integrations (LOCAL, Aramex, Marsool, etc.) — those are fulfillment adapters
- Variant/inheritance relationships between types — fulfillment types are linear; if "DINEIN" exists, it is its own type, not a subtype of `pickup`

## Core decisions

- Fulfillment type is a single string. Core does not expose two-level discrimination (no `type` + `variant`).
- Core pre-registers four types. Plugins and apps add types into the same registry.
- Each type declares a typed `data` schema. Order-level fulfillment data is validated against the schema at write time.
- Type IDs are namespaced. Core types are bare (`'pickup'`); plugin/app types use `<plugin-id>:<name>` (e.g., `'restaurant:dinein'`).
- Adapters consume `order.fulfillmentType` and `order.fulfillmentTypeData` like any other field — no per-type registration on the adapter.

## Registry

The fulfillment type registry is populated at `createCommerce()` startup from three sources:

1. Core's built-in types
2. Inline types declared in `createCommerce({ fulfillments: [ ... ] })` via type factories
3. Types contributed by installed plugins via `plugin({ fulfillments: [ ... ] })`

Each entry has the shape:

```ts
type FulfillmentTypeRegistration = {
  data: StandardSchemaV1           // validated at order write time
  description?: string             // optional, surfaced via the discovery API
}
```

`data` accepts any Standard Schema-compatible value (Zod, Valibot, Arktype, Effect Schema). Examples below use Zod for readability. See [47-validation.md](./47-validation.md).

## Core types

Core pre-registers:

```ts
{
  shipping: {
    data: z.object({
      address: Address,
      carrier: z.string().optional(),
      trackingNumber: z.string().optional(),
    }),
  },
  delivery: {
    data: z.object({
      address: Address,
      coordinates: Coordinates.optional(),
      distanceMeters: z.number().optional(),
      durationSeconds: z.number().optional(),
    }),
  },
  pickup: {
    data: z.object({}).optional(),    // no extra fields by default
  },
  digital: {
    data: z.object({
      downloadUrl: z.string().optional(),
      expiresAt: z.date().optional(),
    }),
  },
}
```

These are normative defaults. Apps that want different shapes for the core types can override them via inline registration (see below) with the same key.

## Plugin contribution

```ts
plugin('restaurant', {
  fulfillments: [
    fulfillmentType('restaurant:dinein', {
      data: z.object({
        tableNumber: z.string(),
      }),
    }),
    fulfillmentType('restaurant:curbside', {
      data: z.object({
        carPlateNumber: z.string(),
        parkingNumber: z.string(),
      }),
    }),
  ],
})
```

Plugin type IDs must begin with `<plugin-id>:` and match the plugin's declared `id`. The startup validator rejects mismatched namespaces with a concrete error.

## Inline contribution

Apps that need one custom type without writing a plugin register inline:

```ts
createCommerce({
  fulfillments: [
    fulfillmentType('my-app:locker', {
      data: z.object({
        lockerNumber: z.string(),
        pin: z.string(),
      }),
    }),
  ],
  plugins: [restaurant()],
})
```

Inline types use a namespace controlled by the app. The convention is `<app-id>:<name>` where `app-id` is configurable in `createCommerce()` (defaults to `'app'`). Collisions with plugin IDs are rejected at startup.

## Order persistence

When fulfillment is active, two fields on `order` carry type information:

```ts
order = {
  ...existing,
  fulfillmentType: text                 // a registered type ID, e.g. 'pickup' or 'restaurant:dinein'
  fulfillmentTypeData: jsonb | null     // validated against the registered type's schema
}
```

The `fulfillmentTypeData` shape is determined entirely by the type's schema. Type narrowing on `fulfillmentType` discriminates `fulfillmentTypeData` automatically (see [60-type-inference.md](./60-type-inference.md)).

`fulfillmentMethod` simply has `type: string` — there is no separate `variant` column. The method's type pins which schema applies to its orders.

## Validation rules

At `createCommerce()` startup:
- Type IDs match their declared source (`<plugin-id>:*` for plugin contributions, configured `<app-id>:*` for inline).
- No duplicate type IDs across sources.
- Core types' schemas may be overridden by inline registration with the same key.

At `fulfillmentMethod.create()` write:
- The `type` field must reference a registered type.

At `orders.checkout()` / `orders.calculate()`:
- The chosen `fulfillmentMethodId` resolves to a method with `type: T`.
- The order's `fulfillmentTypeData` is validated against the schema for `T`.
- If the schema declares required fields and they are missing, the operation throws `CommerceValidationError`.

## Adapter consumption

Fulfillment adapters consume `order.fulfillmentType` and `order.fulfillmentTypeData` like any other order field. They do **not** register types — they only declare which types they handle:

```ts
interface FulfillmentAdapter {
  types: string[]                       // type IDs this adapter handles
  capabilities: FulfillmentCapabilities
  createFulfillment?(ctx): Promise<...> // receives order; reads fulfillmentTypeData
  ...
}
```

An adapter that handles only `'pickup'` ignores `'restaurant:dinein'` orders. An adapter that handles `['pickup', 'restaurant:dinein', 'restaurant:curbside']` switches on `order.fulfillmentType` inside `createFulfillment`:

```ts
async createFulfillment(ctx) {
  if (ctx.order.fulfillmentType === 'restaurant:dinein') {
    const { tableNumber } = ctx.order.fulfillmentTypeData
    return { provider: 'square', note: `Table: ${tableNumber}` }
  }
  // ...
}
```

The variant string is data. The adapter is behavior. They meet in the adapter handler.

## Discovery

A type-listing operation makes the registry visible to admin UIs (e.g., a "create fulfillment method" form):

```ts
commerce.fulfillment.types.list(): Promise<{
  id: string                            // 'pickup' or 'restaurant:dinein'
  description: string | null
  dataSchema: JsonSchema                // Standard Schema → JSON Schema (contract)
}[]>
```

`dataSchema` is always returned as a standard JSON Schema object, converted from the registered Standard Schema (see [47-validation.md](./47-validation.md)) at engine boot. This is part of the SDK contract — admin UIs can render a dynamic form from `dataSchema` without knowing which schema library the type was authored with:

```tsx
function FulfillmentTypeForm({ type }: { type: FulfillmentTypeInfo }) {
  return <JsonSchemaForm schema={type.dataSchema} />   // generic renderer
}
```

The same JSON-Schema contract applies to other registry-shaped surfaces — `commerce.delivery.strategies.list()` (see [55-delivery-pricing.md → Server SDK](./55-delivery-pricing.md#server-sdk)) and `commerce.metadata.operation()` (see [25-server-sdk.md → `metadata`](./25-server-sdk.md#metadata)).

This is the only runtime read against the registry. Validation at write time is core-internal.

## What's not modeled (deliberate)

- **Categories or groups.** No core concept of "pickup-like" types. If a UI needs to group `'pickup'` with `'restaurant:dinein'`, it does so locally — Commerce Kit does not enforce a grouping taxonomy.
- **Inheritance.** No `extends` field. Types are flat.
- **Reservations.** Table reservations, time-slot holds, capacity tracking are separate concerns (likely future plugins). The type system only captures what is on the order at the time it was placed.
- **Per-merchant type overrides.** A merchant cannot disable a registered type — they simply don't expose `fulfillmentMethod` rows for unwanted types.

## Cross-links

- Fulfillment adapter contract: [50-adapter-system.md](./50-adapter-system.md)
- Plugin authoring surface (`fulfillments` contract): [40-plugin-system.md](./40-plugin-system.md)
- Type inference for `fulfillmentTypeData` narrowing: [60-type-inference.md](./60-type-inference.md)
- Order persistence: [20-data-model.md](./20-data-model.md)
- Server SDK for `commerce.fulfillment.*`: [25-server-sdk.md](./25-server-sdk.md)

## Future RFCs

- Table reservation primitive (capacity + time slots + holds)
- Type grouping / category metadata for unified UI rendering
- Per-merchant or per-branch type allowlists
- Variant on a type (`type + variant`) if a real use case emerges that can't be modeled as a separate type
