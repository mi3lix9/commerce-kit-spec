# Calculation Engine

## Purpose

Define the order calculation engine: a pipeline of named steps that produces signed adjustments and derives totals from them. The pipeline is data — a list of step IDs — and may be defined at app config time, per-merchant at runtime, or replaced entirely via custom storage.

Pricing rule implementation details (discount math, tax math, allocation algorithms) live in plugins. The engine in core defines only the contract, the runner, and the persistence hooks.

## Non-goals

- Specific pricing algorithms (proportional allocation, inclusive vs additive tax extraction) — owned by the relevant plugin step
- Pricing rule entities (`Discount`, `Tax`, `ServiceCharge`) — owned by `@commerce-kit/pricing-rules`
- Cart auto-recalculation transport — see [22-cart.md](./22-cart.md)
- Marketplace order splitting (which happens before pricing, once per child order) — see [45-marketplace-mode.md](./45-marketplace-mode.md)

## Core decisions

- The pipeline is a list of step IDs. Steps are registered by plugins.
- Pipeline shape is `string[]` everywhere — in code, in DB, in API payloads.
- Steps emit `Adjustment` records into an append-only list during execution.
- Per-step configuration lives in plugin tables or via multiple registered step variants — not in pipeline entries.
- Pipeline resolution is hookable. By default, the app-level pipeline is used. When `calculation.runtime: true`, per-merchant pipelines stored in a core table override it.
- Money rules (integer minor units, round-half-up) are normative core invariants. Pipelines cannot override them.
- Allocation of order-level adjustments across line items is a fixed core algorithm (proportional to line total, drift rebalanced on the largest line).

## The engine

```ts
type CalculationPipeline = string[]    // ordered list of step IDs

type CalculationStep = {
  id: string                            // 'pricing-rules:discounts'
  type: 'discount' | 'serviceCharge' | 'tax' | 'fulfillment' | 'custom'
  handler: (ctx: StepContext) => Promise<void>
}

type StepContext = {
  state: CalculationState               // current items + accumulated adjustments
  snapshot: Snapshot                    // typed read helpers over state
  emit: (adj: AdjustmentInput) => void
  commerce: CommerceSDK
  request: RequestContext
  input: CalculateInput | CheckoutInput
}

type Adjustment = {
  source: string                        // step ID that emitted this
  kind: 'discount' | 'serviceCharge' | 'tax' | 'fulfillment' | string
  amount: Money                         // signed; discounts are negative
  appliesTo: 'order' | { itemId: string }
  taxable?: boolean                     // contributes to tax basis when true
  metadata?: Record<string, unknown>
}
```

`Snapshot` provides typed read helpers so handlers don't reach into raw adjustments:

```ts
type Snapshot = {
  readonly subtotal: Money
  readonly adjustments: readonly Adjustment[]
  totalOf(kind: string, filter?: (a: Adjustment) => boolean): number
  byKind(kind: string): readonly Adjustment[]
}
```

## Lifecycle

The pipeline runs inside three operations:

| Operation | Pipeline runs | Result persisted |
|---|---|---|
| `commerce.orders.calculate(...)` | yes | no — pure read |
| `commerce.orders.checkout(...)` | yes | yes (inside transaction) |
| `commerce.orders.recalculate({ id })` | yes (only while order is in `draft`) | yes |

Inside the operation:

```
orders.calculate / checkout / recalculate
│
├─ 'orders:<op>:before' handlers
│
├─ Resolve items, build initial state (subtotal, empty adjustments)
├─ Resolve pipeline via 'calculation:resolvePipeline' hook
│
├─ For each stepId in pipeline:
│  ├─ Look up registered step by ID
│  ├─ Run step.handler(ctx) — handler calls emit() N times
│  └─ Append emitted adjustments to state
│
├─ Finalize: derive totals, allocate order-level adjustments to line items
├─ Persist (checkout/recalculate only)
│
└─ 'orders:<op>:after' handlers
```

Handlers run **sequentially and awaited**. Throwing inside any step aborts the calculation and (in checkout) rolls back the transaction.

## Step registration

Plugins register named steps. The step ID convention is `<plugin-id>:<name>`.

```ts
plugin('pricing-rules', {
  calculation: {
    steps: {
      'pricing-rules:discounts': {
        type: 'discount',
        handler: async ({ snapshot, emit, commerce, request }) => {
          const rules = await commerce.discounts.list({ where: { active: true } })
          for (const rule of rules) {
            emit({
              kind: 'discount',
              source: `pricing-rules:discounts:${rule.id}`,
              amount: computeDiscountAmount(rule, snapshot),
              appliesTo: rule.appliesTo,
            })
          }
        },
      },

      'pricing-rules:taxes': {
        type: 'tax',
        handler: async ({ snapshot, emit, commerce, request }) => {
          const rules = await commerce.taxes.list({ where: { active: true } })
          const taxableBase =
            snapshot.subtotal
            + snapshot.totalOf('discount')
            + snapshot.totalOf('serviceCharge', a => a.taxable === true)

          for (const rule of rules) {
            emit({
              kind: 'tax',
              source: `pricing-rules:taxes:${rule.id}`,
              amount: { amount: Math.round(taxableBase * rule.basisPoints / 10000), currency: 'SAR' },
              appliesTo: 'order',
              metadata: { rate: rule.basisPoints, type: rule.type },
            })
          }
        },
      },
    },
  },
})
```

### Multiple variants of one step

Plugins that need parameterized steps register multiple IDs:

```ts
calculation: {
  steps: {
    'tax:VAT-SA': { type: 'tax', handler: makeTaxHandler({ rate: 1500, jurisdiction: 'SA' }) },
    'tax:VAT-EU': { type: 'tax', handler: makeTaxHandler({ rate: 2000, jurisdiction: 'EU' }) },
  },
}
```

Pipelines then reference whichever variant they want.

## Core-registered steps

Core itself registers a small set of steps under the `core:*` namespace. These are auto-registered when the corresponding feature slot is active and absent when the slot is dormant.

| Step ID | Registered when | Type | What it emits |
|---|---|---|---|
| `core:delivery-fee` | `deliveries: [...]` is non-empty | `fulfillment` | One `appliesTo: 'order'` adjustment from the order's `deliveryMethod` pricing strategy. Throws on `below_min_amount` / `out_of_range` / `no_zone_match`. See [55-delivery-pricing.md](./55-delivery-pricing.md). |

The default pipeline auto-injects active `core:*` steps in a fixed order:

```
pricing-rules:discounts
pricing-rules:serviceCharges
core:delivery-fee
pricing-rules:taxes
```

Discounts and service charges come first so they can shift the taxable base. Delivery fee is taxable too — it sits before tax. Apps that need a different order set `calculation.pipeline` explicitly.

When the `delivery` slot is dormant, `core:delivery-fee` is neither registered nor auto-injected. Referencing it in an explicit pipeline raises a startup error.

Future `core:*` steps (e.g., `core:rounding`, `core:loyalty-points`) will follow the same auto-injection rule — present only when their feature slot is active.

## Pipeline sources, in priority order

1. **Per-merchant DB row** (when `calculation.runtime: true` and a row exists for the current `request.merchantId`)
2. **App-level default** from `createCommerce({ calculation: { pipeline } })`
3. **Plugin contribution order** — if neither of the above is set, all plugin steps run in plugin declaration order

Resolution happens at calculation time via the `calculation:resolvePipeline` hook. Apps and plugins can override the chain by implementing their own resolver.

## App config

```ts
createCommerce({
  database: ...,
  payments: [moyasar({ ... })],
  plugins: [pricingRulesPlugin()],

  calculation: {
    pipeline: [
      'pricing-rules:discounts',
      'pricing-rules:serviceCharges',
      'pricing-rules:taxes',
    ],
    runtime: true,
  },
})
```

| Field | Effect |
|---|---|
| `pipeline?: string[]` | The default pipeline for any request without a per-merchant override |
| `runtime?: boolean` | When `true`, materializes the per-merchant pipeline table and exposes `commerce.calculation.{set,get}Pipeline` |

`runtime: true` requires `tenancy.merchants: true`. Without merchants there's nothing to scope a per-merchant override to — install-time validation rejects it.

## Per-merchant runtime override

When `calculation.runtime: true` is set:

- Core materializes a `merchantCalculationPipelines` table:
  ```
  merchantCalculationPipelines = {
    merchantId: text (PK, FK → merchants)
    pipeline: jsonb     // string[]
    updatedAt: timestamp
  }
  ```
- Core adds `commerce.calculation.*` to the SDK:
  ```ts
  commerce.calculation.getPipeline({ merchantId }): Promise<{ pipeline: string[] } | null>
  commerce.calculation.setPipeline({ merchantId, pipeline: string[] }): Promise<void>
  commerce.calculation.resetPipeline({ merchantId }): Promise<void>
  ```
- Core mounts HTTP routes:
  ```
  GET    /merchants/:id/calculation-pipeline
  PUT    /merchants/:id/calculation-pipeline
  DELETE /merchants/:id/calculation-pipeline
  ```
- The built-in resolver hook reads the merchant's row at calculation time, falling back to the app default if no row exists.

Authorization: by default, a request actor may only `set` or `reset` the pipeline for a merchant they own. Apps can layer additional checks via `'orders:calculate:before'` or by replacing the route.

## Custom storage via the resolver hook

The `calculation:resolvePipeline` hook is always exposed. Apps that want non-DB storage (Redis, file, remote config service) set `runtime: false` and install a plugin that hooks the resolver:

```ts
const remoteConfigPipelines = plugin('remote-config-pipelines', {
  on: {
    'calculation:resolvePipeline': async ({ request, default: def }) => {
      if (!request.merchantId) return def
      const remote = await fetchPipelineFromConfigService(request.merchantId)
      return remote ?? def
    },
  },
})
```

The hook receives the current default and returns either it or a replacement. Returning `undefined` is equivalent to returning the default.

## Validation

Core validates pipelines at two boundaries:

1. **At `createCommerce()` startup** — every step ID in `calculation.pipeline` must be registered by some installed plugin. Unknown IDs are a startup error.
2. **At `setPipeline()` write** — same check applies before the row is written. Unknown IDs return a `CommerceValidationError`.

This makes DB-stored pipelines safe: removing a plugin causes startup to fail loudly, rather than silently producing wrong totals.

## Finalization and allocation

After all steps run, core finalizes the result:

- `subtotal` = sum of line totals (no adjustments)
- `discountTotal`, `serviceChargeTotal`, `taxTotal`, `fulfillmentTotal` = `snapshot.totalOf(kind)` for each canonical kind
- `total` = `subtotal + sum(all adjustments where !metadata.taxInclusive)`
- For each `appliesTo: 'order'` adjustment, core allocates its amount across line items proportionally to line total. Rounding drift is corrected by adding/subtracting one minor unit on the largest line.
- For each `appliesTo: { itemId }` adjustment, the amount is pinned to that line item only.

The persisted result on `order` includes the full `adjustments: jsonb` array as the audit trail. Derived totals are stored as columns for query convenience but the adjustments array is the source of truth.

### Inclusive tax handling

Steps may emit adjustments with `metadata.taxInclusive: true` to signal that the amount is already part of the prices (extracted from the base, not added on top). These adjustments:

- DO contribute to `taxTotal` (so reporting shows the tax portion of the inclusive price)
- DO NOT contribute to `total` (the customer-facing price didn't increase — the tax was always there)

This is the only place where engine math differs based on adjustment metadata. All other adjustment kinds add or subtract from `total` based on their signed `amount`.

## `explain()` — pricing introspection

Every calculation result exposes a side-effect-free `explain()` that traces which steps ran, which adjustments they emitted, and which rules were considered but skipped. `commerce.calculation.explain({ items, merchantId? })` runs the pipeline in a dry-run mode that returns the same trace without persisting the result. Both are intended for debugging — production code paths should not depend on the trace shape.

```ts
const result = await commerce.orders.calculate({ items })
const trace = result.explain()
// {
//   pipelineSource: { kind: 'merchantOverride', merchantId: 'm_123' },
//   pipeline: ['pricing-rules:discounts', 'core:delivery-fee', 'pricing-rules:taxes'],
//   steps: [
//     {
//       id: 'pricing-rules:discounts',
//       durationMs: 0.4,
//       emitted: [
//         { kind: 'discount', amount: -500, ruleId: 'r_summer' },
//       ],
//       skipped: [
//         { ruleId: 'r_xmas', reason: 'cart subtotal below minimum' },
//         { ruleId: 'r_loyalty', reason: 'customer not enrolled' },
//       ],
//     },
//     {
//       id: 'core:delivery-fee',
//       durationMs: 0.1,
//       emitted: [{ kind: 'fulfillment', amount: 1500, ruleId: 'flat-riyadh' }],
//     },
//     {
//       id: 'pricing-rules:taxes',
//       durationMs: 0.2,
//       emitted: [{ kind: 'tax', amount: 225, ruleId: 'vat-15' }],
//     },
//   ],
// }

// Hypothetical — does not persist
const dryRun = await commerce.calculation.explain({ items, merchantId: 'm_123' })
```

Trace recording is gated behind a development flag so production calculations stay zero-overhead. The trace shape is part of the SDK contract — third-party debug UIs can render against it.

## What runs without any plugin

If no plugin registers calculation steps and no pipeline is configured:

- The pipeline is empty.
- No adjustments are emitted.
- `total = subtotal`.

This is a valid configuration for a zero-tax, zero-discount store and requires no install or config.

## Cross-links

- Pricing rule entities and default step implementations: [44-pricing-rules-plugin.md](./44-pricing-rules-plugin.md)
- Money rules and rounding invariants: [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Plugin authoring surface and `calculation.steps` contract: [40-plugin-system.md](./40-plugin-system.md)
- Hook context shape and `:before`/`:after` semantics: [42-hooks.md](./42-hooks.md)
- Per-tenant context resolution: [12-tenancy.md](./12-tenancy.md)
- Server SDK shape for `commerce.calculation.*` and `commerce.orders.*`: [25-server-sdk.md](./25-server-sdk.md)

## Future RFCs

- Step versioning for safe schema evolution
- Step lifecycle hooks (`step:before`, `step:after`) for tracing and validation
- Pipeline templates — named pipelines that can be applied to merchants in bulk
- Dry-run mode for testing pipeline changes before applying them
