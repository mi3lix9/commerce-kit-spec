# Validation

## Purpose

Define the schema validation contract used everywhere Commerce Kit accepts a schema value: plugin `operations.input`, fulfillment type `data`, delivery pricing strategy `settings`, and any future schema-shaped surface. The contract is library-agnostic — host apps and plugin authors use whichever schema library they already standardized on.

## Non-goals

- Defining a Commerce Kit schema library or DSL. Commerce Kit declares tables through `table()` and column helpers — see [40-plugin-system.md](./40-plugin-system.md).
- Defining the `CommerceValidationError` shape — owned by [15-errors.md](./15-errors.md).
- HTTP error response shape — owned by [15-errors.md](./15-errors.md) and [80-framework-adapters.md](./80-framework-adapters.md).
- Serialized schema projections for admin UIs (`dataSchema: JsonSchema`, `settingsSchema: JsonSchema`) — those are output projections, not validation inputs.

## Core decisions

- Every Commerce Kit surface that accepts a schema accepts a [Standard Schema](https://standardschema.dev) value (`@standard-schema/spec`).
- Standard Schema is the only validation contract surfaced by Commerce Kit. Commerce Kit does not bundle Zod, Valibot, Arktype, or any concrete library.
- Host apps and plugin authors bring their own library. Zod, Valibot, Arktype, Effect Schema, and other libraries that emit `~standard` are all supported equally.
- Examples in the docs use Zod for familiarity; this is documentation style, not a runtime requirement.
- Validation failures throw `CommerceValidationError` with `issues` normalized from the Standard Schema result.

## Why Standard Schema

The *guest in your stack* principle applies to validation libraries the same way it applies to ORMs and frameworks. Hard-coding Zod would force every host app onto the library Commerce Kit happens to pick. Standard Schema is a minimal interop spec, already supported by Zod, Valibot, Arktype, and Effect Schema. Accepting `StandardSchemaV1` lets every host keep the library they already use.

## What Standard Schema gives Commerce Kit

- A `~standard.validate(input)` function returning either `{ value }` or `{ issues }`.
- `InferInput<TSchema>` and `InferOutput<TSchema>` for type inference at handler boundaries.
- Vendor metadata (`version`, `vendor`).

It does not define schema composition (`.extend`, `.merge`, `.partial`, `.pick`). Those remain library-specific. Commerce Kit never composes a caller-supplied schema.

## Required surfaces

Every Commerce Kit surface listed below accepts a Standard Schema. A value passed that does not satisfy `StandardSchemaV1` is rejected at `createCommerce()` initialization with `CommerceConfigError`.

| Surface | Where declared | What it validates |
|---|---|---|
| `operations.<key>.input` | plugin `operations` block — see [40-plugin-system.md](./40-plugin-system.md) | the operation's `input` argument on every `commerce.*` call |
| `fulfillmentType('id', { data })` | plugin `fulfillments` block or inline at `createCommerce()` — see [52-fulfillment-types.md](./52-fulfillment-types.md) | `fulfillmentTypeData` at order write time |
| `deliveryPricingStrategy({ settings })` | `deliveryPricing: [...]` slot — see [55-delivery-pricing.md](./55-delivery-pricing.md) | strategy `settings` on `delivery.methods.create` / `update`, plus startup validation of every persisted method's stored settings |
| webhook event payload | adapter `verifyWebhook` contract — see [50-adapter-system.md](./50-adapter-system.md) | the verified event passed to plugin `webhooks` handlers |

Additional surfaces added by future RFCs follow the same rule.

## Validation runtime

Commerce Kit's validation primitive accepts a schema, an input value, and a surface tag. It calls `schema['~standard'].validate(input)` and inspects the return value:

- If the result is a `Promise`, it is awaited.
- Otherwise the synchronous value is used directly.

Sync schemas stay on the synchronous path; async schemas are awaited. Both forms are valid at every surface. Hot paths that know their schemas are synchronous may type the result accordingly.

On `{ issues }`, the primitive throws `CommerceValidationError` carrying:

- a normalized `issues: ValidationIssue[]` array — see *Error mapping* below
- a surface tag identifying the failing boundary, exposed through `error.data.surface`

The surface tag uses these stable strings:

| Tag | Boundary |
|---|---|
| `operation:<plugin-id>:<operation-key>` | plugin operation input |
| `operation:core:<operation-key>` | core operation input |
| `fulfillment-type:<type-id>:data` | fulfillment data at order write time |
| `delivery-pricing:<strategy-tag>:settings` | delivery pricing strategy settings |
| `webhook:<adapter-id>:<event-type>` | webhook event payload |

## Error mapping

`CommerceValidationError` defines its own `ValidationIssue` shape — see [15-errors.md](./15-errors.md). Standard Schema's issue shape is normalized into that shape at validation time:

```ts
type ValidationIssue = {
  path: (string | number)[]
  message: string
  code: string
}
```

Normalization rules:

- Standard Schema's `path` entries (which may be `PropertyKey | { key: PropertyKey }`) are flattened to `(string | number)[]`. Object-form entries use their `key`. Non-string, non-number keys are stringified.
- `message` is preserved verbatim from the Standard Schema issue.
- `code` defaults to the vendor's code when present (Zod's `code`, Valibot's `type`, etc.); when absent, `code` is `'invalid'`.

The HTTP envelope shape and `instanceof` checks on the client are unchanged — they only see the normalized `ValidationIssue[]`.

## Type inference

Operation handlers, fulfillment data validators, and delivery pricing strategies receive inputs typed as `InferOutput<typeof schema>`. The `input` field exposed at the call site is typed as `InferInput<typeof schema>`. A plugin author writing:

```ts
operations: {
  'coupons:create': {
    input: z.object({ code: z.string(), discountValue: z.number().int().positive() }),
    handler: async ({ input }) => {
      // input is { code: string; discountValue: number }
    },
  },
}
```

gets `input` narrowed automatically. Swapping `z.object(...)` for Valibot's equivalent (`v.object(...)`) does not change the handler body — both emit Standard Schemas, and both produce the same narrowed type at the consumption site.

## Forbidden behavior

- Passing a value that is not a Standard Schema to a surface that accepts a schema. Detected at `createCommerce()` initialization.
- Coercing a Standard Schema's `issues` array into a different shape inside a plugin instead of letting `CommerceValidationError` normalize.
- Bundling a concrete schema library as a runtime dependency of `commerce-kit` core.
- Composing or mutating a caller-supplied schema inside the core (extend, merge, partial). Type information flows out via `InferOutput`; the schema object is read, never modified.

## Library examples

The following are all valid Standard Schemas accepted at every surface. Pick whichever your host app already uses.

```ts
// Zod 3.24+
import { z } from 'zod'
const input = z.object({ code: z.string() })

// Valibot
import * as v from 'valibot'
const input = v.object({ code: v.string() })

// Arktype
import { type } from 'arktype'
const input = type({ code: 'string' })

// Effect Schema
import { Schema } from 'effect'
const input = Schema.Struct({ code: Schema.String })
```

Documentation examples in [40-plugin-system.md](./40-plugin-system.md), [52-fulfillment-types.md](./52-fulfillment-types.md), and [55-delivery-pricing.md](./55-delivery-pricing.md) use Zod for readability. The contract is the Standard Schema interface, not Zod itself.

## Packaging

`@standard-schema/spec` is a peer dependency of `commerce-kit`, declared in `peerDependencies` and shipping types only — it contributes no runtime code. See [70-packages-and-monorepo.md](./70-packages-and-monorepo.md).

## Cross-links

- `CommerceValidationError` class and `ValidationIssue` shape: [15-errors.md](./15-errors.md)
- Plugin `operations.input`: [40-plugin-system.md](./40-plugin-system.md)
- Fulfillment type `data`: [52-fulfillment-types.md](./52-fulfillment-types.md)
- Delivery pricing strategy `settings`: [55-delivery-pricing.md](./55-delivery-pricing.md)
- Adapter `verifyWebhook` event normalization: [50-adapter-system.md](./50-adapter-system.md)
- Calculation pipeline validation at write time: [35-calculation-engine.md](./35-calculation-engine.md)

## Future RFCs

- Additional schema-accepting surfaces beyond the current set.
- Helpers for issue formatting if host-app demand grows beyond the normalized `ValidationIssue[]`.
- Serialized JSON Schema projection guarantees for `dataSchema` / `settingsSchema` outputs (currently a documented best-effort).
