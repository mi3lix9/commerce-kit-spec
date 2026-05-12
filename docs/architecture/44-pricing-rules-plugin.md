# Pricing Rules Plugin

## Purpose

Define `@commerce-kit/pricing-rules`: the default plugin that contributes the three canonical calculation steps (`pricing-rules:discounts`, `pricing-rules:serviceCharges`, `pricing-rules:taxes`) to the calculation engine, along with the three rule entities that drive them.

This plugin is the reference implementation of the engine's step contract. Apps that want different algorithms (e.g., jurisdiction-specific tax math, custom discount stacking) can swap in their own plugins exposing the same step IDs — or different ones — without touching core.

## Non-goals

- Coupon codes — see `@commerce-kit/coupons` (planned). Coupons emit into `pricing-rules:discounts` slot via their own plugin step, not by reusing this plugin's `discount` entity.
- The calculation engine, step registry, or pipeline runner — see [35-calculation-engine.md](./35-calculation-engine.md).
- Money primitives and rounding — see [30-pricing-and-calculations.md](./30-pricing-and-calculations.md).

## Core decisions

- Three separate entities (`discount`, `tax`, `serviceCharge`) instead of one polymorphic table. Each has its own typed config shape.
- All three entities use hierarchical tenancy: `merchant().optional()` and `branch().optional()`. A rule may apply platform-wide, per-merchant, or per-branch, with most specific match winning.
- Tax math handles both inclusive and additive types in a single step. The plugin owns the inclusive-extraction algorithm.
- Service charges declare `taxable: boolean`. The tax step includes only taxable SCs in its basis.
- Each step is independent — apps may install one, two, or all three by composing factory functions.

## Plugin shape

```ts
import { pricingRules } from '@commerce-kit/pricing-rules'

createCommerce({
  plugins: [
    pricingRules.discounts(),
    pricingRules.serviceCharges(),
    pricingRules.taxes(),
  ],

  calculation: {
    pipeline: [
      'pricing-rules:discounts',
      'pricing-rules:serviceCharges',
      'pricing-rules:taxes',
    ],
  },
})
```

Or install all three at once with the bundled plugin:

```ts
import { pricingRulesPlugin } from '@commerce-kit/pricing-rules'

createCommerce({
  plugins: [pricingRulesPlugin()],
})
```

The bundled plugin registers all three steps. Each sub-plugin registers exactly one.

## Entities

### `discount`

```ts
discount = {
  id: string
  name: string
  description: string | null

  type: 'percentage' | 'fixed'
  value: number                                // basis points for percentage; minor units for fixed
  maxDiscountAmount: number | null             // cap for percentage discounts

  appliesTo: Targeting
  startsAt: Date | null
  endsAt: Date | null
  active: boolean

  metadata: Record<string, unknown>

  // Tenancy columns (dormant when tenancy off)
  merchant: merchant().optional()
  branch: branch().optional()

  createdAt: Date
  updatedAt: Date
}
```

`type: 'percentage'` with `value: 1000` means 10.00%. `value: 1500` is 15.00%.
`type: 'fixed'` with `value: 500` means 5.00 in minor units.

### `tax`

```ts
tax = {
  id: string
  name: string

  type: 'inclusive' | 'additive'
  basisPoints: number                          // 1500 = 15.00%

  appliesTo: Targeting
  includesTaxableServiceCharges: boolean       // default true
  active: boolean

  metadata: Record<string, unknown>

  merchant: merchant().optional()
  branch: branch().optional()

  createdAt: Date
  updatedAt: Date
}
```

`inclusive` taxes are extracted from the targeted base (the displayed prices already contain the tax). `additive` taxes are added on top.

### `serviceCharge`

```ts
serviceCharge = {
  id: string
  name: string

  type: 'percentage' | 'fixed'
  value: number                                // basis points for percentage; minor units for fixed
  scope: 'order' | 'lineItem'                  // applies once to the order, or per-line

  appliesTo: Targeting
  taxable: boolean
  active: boolean

  metadata: Record<string, unknown>

  merchant: merchant().optional()
  branch: branch().optional()

  createdAt: Date
  updatedAt: Date
}
```

`scope: 'order'` charges once per order (e.g., a platform fee). `scope: 'lineItem'` charges per line item, allocated by the plugin (e.g., a packaging fee per item).

## Targeting

All three entities share a single `Targeting` discriminated union:

```ts
type Targeting =
  | { kind: 'order' }                          // applies to the whole order/subtotal
  | { kind: 'categories'; categoryIds: string[] }
  | { kind: 'products'; productIds: string[] }
  | { kind: 'variants'; variantIds: string[] }
```

A rule with `appliesTo.kind === 'order'` uses the full subtotal as its base. A rule with item-level targeting uses only the subset of line items matching the targeting set.

When a rule targets a subset, its emitted adjustment uses `appliesTo: 'order'` if the targeting still represents an order-level concept (e.g., 10% off products in category X = one order-level adjustment whose amount is computed from those items' subtotal) or `appliesTo: { itemId }` if the rule is intrinsically per-line (e.g., a per-item packaging fee).

The plugin chooses based on `serviceCharge.scope`:

| Rule | Emitted as |
|---|---|
| Discount with `kind: 'order'` | one `appliesTo: 'order'` adjustment |
| Discount with item-level targeting | one `appliesTo: 'order'` adjustment (amount computed from matching items) |
| ServiceCharge with `scope: 'order'` | one `appliesTo: 'order'` adjustment |
| ServiceCharge with `scope: 'lineItem'` | N `appliesTo: { itemId }` adjustments |
| Tax | one `appliesTo: 'order'` adjustment per tax rule |

## Step implementations

### `pricing-rules:discounts`

```ts
async ({ snapshot, emit, commerce, request }) => {
  const rules = await commerce.discounts.listApplicable({
    at: new Date(),
    request,
  })

  for (const rule of rules) {
    const targetBase = resolveTargetBase(rule.appliesTo, snapshot.items)
    if (targetBase <= 0) continue

    const amount = rule.type === 'percentage'
      ? Math.round(targetBase * rule.value / 10_000)
      : Math.min(rule.value, targetBase)

    const capped = rule.maxDiscountAmount != null
      ? Math.min(amount, rule.maxDiscountAmount)
      : amount

    emit({
      kind: 'discount',
      source: `pricing-rules:discounts:${rule.id}`,
      amount: { amount: -capped, currency: snapshot.currency },
      appliesTo: 'order',
      metadata: { ruleId: rule.id, type: rule.type, value: rule.value },
    })
  }
}
```

`commerce.discounts.listApplicable` filters by:
- `active: true`
- `startsAt <= now` (or null)
- `endsAt > now` (or null)
- hierarchical tenancy match via the request's `merchantId` / `branchId`

### `pricing-rules:serviceCharges`

```ts
async ({ snapshot, emit, commerce, request }) => {
  const rules = await commerce.serviceCharges.listApplicable({ request })

  for (const rule of rules) {
    if (rule.scope === 'order') {
      const targetBase = resolveTargetBase(rule.appliesTo, snapshot.items)
      const amount = rule.type === 'percentage'
        ? Math.round(targetBase * rule.value / 10_000)
        : rule.value
      emit({
        kind: 'serviceCharge',
        source: `pricing-rules:serviceCharges:${rule.id}`,
        amount: { amount, currency: snapshot.currency },
        appliesTo: 'order',
        taxable: rule.taxable,
        metadata: { ruleId: rule.id, scope: rule.scope },
      })
    } else {
      // scope === 'lineItem'
      const targetItems = filterItemsByTargeting(rule.appliesTo, snapshot.items)
      for (const item of targetItems) {
        const lineBase = item.lineTotal
        const amount = rule.type === 'percentage'
          ? Math.round(lineBase * rule.value / 10_000)
          : rule.value
        emit({
          kind: 'serviceCharge',
          source: `pricing-rules:serviceCharges:${rule.id}`,
          amount: { amount, currency: snapshot.currency },
          appliesTo: { itemId: item.id },
          taxable: rule.taxable,
          metadata: { ruleId: rule.id, scope: rule.scope },
        })
      }
    }
  }
}
```

### `pricing-rules:taxes`

The tax step is the most complex because it handles both inclusive and additive types and respects `taxable` service charges.

```ts
async ({ snapshot, emit, commerce, request }) => {
  const rules = await commerce.taxes.listApplicable({ request })

  // Build the taxable base, respecting discounts and taxable service charges
  const discountTotal = snapshot.totalOf('discount')                                     // negative
  const taxableServiceChargeTotal = snapshot.totalOf('serviceCharge', a => a.taxable === true)

  for (const rule of rules) {
    const targetSubtotal = resolveTargetBase(rule.appliesTo, snapshot.items)
    const discountShare = allocateProportional(discountTotal, targetSubtotal, snapshot.subtotal)
    const scShare = rule.includesTaxableServiceCharges
      ? allocateProportional(taxableServiceChargeTotal, targetSubtotal, snapshot.subtotal)
      : 0
    const base = Math.max(0, targetSubtotal + discountShare + scShare)

    const amount = rule.type === 'inclusive'
      ? extractInclusive(base, rule.basisPoints)
      : addAdditive(base, rule.basisPoints)

    emit({
      kind: 'tax',
      source: `pricing-rules:taxes:${rule.id}`,
      amount: { amount, currency: snapshot.currency },
      appliesTo: 'order',
      metadata: {
        ruleId: rule.id,
        type: rule.type,
        rate: rule.basisPoints,
        base,
      },
    })
  }
}

// Inclusive: amount is part of the base; extracts the tax portion
function extractInclusive(base: number, basisPoints: number): number {
  return Math.round(base * basisPoints / (basisPoints + 10_000))
}

// Additive: tax is added on top of the base
function addAdditive(base: number, basisPoints: number): number {
  return Math.round(base * basisPoints / 10_000)
}
```

### Inclusive vs additive — what the totals look like

For a 100.00 base with 15% tax:

| Type | Subtotal | Tax | Total | Customer sees |
|---|---|---|---|---|
| Additive | 10000 | 1500 | 11500 | "100.00 + 15.00 tax = 115.00" |
| Inclusive | 10000 | 1304 (extracted) | 10000 | "100.00 incl. 13.04 tax" |

Both are valid. The plugin emits the right `amount` for each — for inclusive, the tax is reported but doesn't increase the total because the base already contains it. The engine handles this naturally: the `taxTotal` reflects the extracted portion, but the `total` equals `subtotal + adjustments` only when none of the adjustments are inclusive. For inclusive taxes, the plugin's emit math accounts for this so totals add up.

Implementation note: inclusive tax emits a tax adjustment whose `amount` is the extracted portion, AND a "synthetic" zero-sum adjustment on the subtotal would be required to make `subtotal + adjustments = total` correctly. To avoid this complexity, the inclusive tax step instead records the extracted tax in `metadata.extractedFrom` and the engine knows not to add inclusive taxes to the total. See [35-calculation-engine.md](./35-calculation-engine.md) for the engine's handling of `metadata.taxInclusive`.

## SDK surface

The plugin contributes three namespaces to `commerce.*`:

```ts
commerce.discounts.{list, get, create, update, archive, listApplicable}
commerce.taxes.{list, get, create, update, archive, listApplicable}
commerce.serviceCharges.{list, get, create, update, archive, listApplicable}
```

Each `listApplicable` is the runtime read used by the corresponding step. It filters by:
- `active: true`
- date range (start/end)
- hierarchical tenancy match
- targeting compatibility with current cart (optional — the plugin can return all and let the step filter)

CRUD operations follow the standard server SDK shape from [25-server-sdk.md](./25-server-sdk.md).

## HTTP routes

When at least one of the three sub-plugins is installed, the corresponding routes mount:

```
GET    /discounts            POST   /discounts
GET    /discounts/:id        PATCH  /discounts/:id
                             DELETE /discounts/:id   (archive)

GET    /taxes                POST   /taxes
GET    /taxes/:id            PATCH  /taxes/:id
                             DELETE /taxes/:id

GET    /service-charges      POST   /service-charges
GET    /service-charges/:id  PATCH  /service-charges/:id
                             DELETE /service-charges/:id
```

Tenancy scoping (the `merchant().optional()` and `branch().optional()` columns) is enforced automatically. A merchant admin sees only their own rules; an app admin sees all.

## Tenancy support

```ts
support: {
  merchants: 'optional',
  branches: 'optional',
}
```

All three entities use `merchant().optional()` and `branch().optional()` columns. In simple-store mode, all rules are platform-wide. In multi-merchant mode, rules can be platform / merchant / branch with hierarchical precedence (most specific match wins).

## Replacement and swap-in

Any of the three steps can be replaced by an app-supplied plugin that registers the same step ID. Core's collision policy requires that step IDs be unique, so the replacing plugin must first uninstall or be installed instead of the default.

A safer pattern for custom math: register your own step ID (e.g., `my-app:custom-taxes`) and reference it in the pipeline instead of `pricing-rules:taxes`. Both plugins can coexist; only the referenced one runs.

```ts
createCommerce({
  plugins: [
    pricingRules.discounts(),
    pricingRules.serviceCharges(),
    customTaxesPlugin(),                          // registers 'my-app:custom-taxes'
  ],
  calculation: {
    pipeline: [
      'pricing-rules:discounts',
      'pricing-rules:serviceCharges',
      'my-app:custom-taxes',                      // ← swap point
    ],
  },
})
```

## Future RFCs

- Condition expressions on rules (min order amount, customer tags, time-of-day)
- Stackable vs exclusive discount strategies
- Rule priority / ordering within the same step
- Cross-rule constraints (e.g., "this discount cannot combine with that tax exemption")
- Tax jurisdictions and zone matching beyond simple targeting

## Cross-links

- Calculation engine (step contract, pipeline, runtime): [35-calculation-engine.md](./35-calculation-engine.md)
- Money invariants and adjustment audit trail: [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Plugin authoring (`plugin()`, `table()`, `calculation.steps`): [40-plugin-system.md](./40-plugin-system.md)
- Tenancy columns and hierarchical reads: [12-tenancy.md](./12-tenancy.md)
- Server SDK conventions: [25-server-sdk.md](./25-server-sdk.md)
