# Tenancy

## Purpose

Define how Commerce Kit handles multi-merchant and multi-branch deployments. Tenancy is a core capability with dormant activation: off by default (simple store), on when declared in `createCommerce()`. Plugins automatically work in both modes through declared `support` and column helpers.

## Non-goals

- Marketplace UX (cross-merchant storefronts, shared cart) — see [45-marketplace-mode.md](./45-marketplace-mode.md)
- Plugin authoring surface beyond tenancy concerns — see [40-plugin-system.md](./40-plugin-system.md)
- Per-tenant adapter configuration — out of scope for v1

## Core decisions

- Tenancy is a core concern, not a plugin. It cannot be reinvented by plugins.
- Two tenancy axes: **merchants** and **branches**. Each independently optional.
- `branches` requires `merchants` (a branch without an owning merchant has no business model).
- Tenancy is **dormant** when not declared — zero columns added, zero query overhead, simple-store DX unchanged.
- Tenancy participation per table is expressed through **nullable foreign keys**, not abstract scope vocabulary.
- Plugins declare their tenancy requirements through a `support` block. Mismatches with the active tenancy config fail at install time with concrete errors.
- Reads against tables that participate in tenancy use **hierarchical resolution**: most specific match wins (branch > merchant > platform).
- Writes derive tenancy values from the request context unless explicitly overridden.

## Config shape

```ts
createCommerce({
  database: drizzleAdapter(db, { schema }),
  payment: { moyasar: ... },

  // Simple store — tenancy not declared
})

createCommerce({
  ...,
  tenancy: {
    merchants: true,
  },
})

createCommerce({
  ...,
  tenancy: {
    merchants: true,
    branches: true,
  },
})
```

`tenancy.branches: true` implies `tenancy.merchants: true`. Setting `branches: true` without `merchants: true` is a compile-time error.

## Column helpers

Plugins (and core) declare tenancy participation per table using `merchant()` and `branch()` column builders. The shape of the row tells Commerce Kit everything it needs to know.

```ts
import { plugin, table, text, integer, merchant, branch } from 'commerce-kit'

// Hierarchical — can exist at platform, merchant, or branch level
const coupons = table('coupons', {
  code: text().unique(),
  discountValue: integer(),
  merchant: merchant().optional(),
  branch: branch().optional(),
})

// Per-merchant only
const merchantSettings = table('merchantSettings', {
  vatNumber: text(),
  merchant: merchant(),   // required
})

// Per-branch only
const businessHours = table('businessHours', {
  periods: jsonb(),
  branch: branch(),   // required (implies merchant resolution via branch)
})

// Truly global — no tenancy participation
const auditLog = table('auditLog', {
  operation: text(),
  // no merchant(), no branch()
})
```

### How shape maps to behavior

| `merchant()` | `branch()` | Row participation |
|---|---|---|
| absent | absent | platform-wide |
| optional | absent | merchant-or-platform |
| required | absent | merchant-pinned |
| optional | optional | hierarchical (platform / merchant / branch) |
| required | optional | merchant-pinned with optional branch specialization |
| (any) | required | branch-pinned |

### What core does with the columns

1. **Tenancy off:** `merchant()` and `branch()` resolve to nothing. Columns are omitted from the materialized schema.
2. **Tenancy on:** columns become foreign keys to the tenancy tables, with `NOT NULL` or `NULL` matching the `.optional()` flag.
3. **Writes:** if a tenancy column is declared and not explicitly provided in input, core fills it from `request.merchantId` / `request.branchId`. If the request has no value and the column is required, the write fails with a clear error.
4. **Reads via `findUnique` / `findFirst`:** hierarchical OR — `WHERE (merchant IS NULL OR merchant = $m) AND (branch IS NULL OR branch = $l)` — most specific match wins.
5. **Reads via `findMany` / `list`:** returns all visible rows for the current context, ordered most-specific first by default.

## Hierarchical resolution rule

For tables where both `merchant()` and `branch()` are optional, multiple rows may match a single context. The resolution order is fixed:

1. **Branch match** wins if a row exists with the current `merchant` and `branch` filled.
2. **Merchant match** otherwise, if a row exists with the current `merchant` and `branch` null.
3. **Platform match** otherwise, if a row exists with both null.

`findUnique` returns exactly the winner. `findMany` returns all matches ordered by precedence. This is a documented invariant and applies uniformly to every hierarchical table.

## Plugin `support` block

Plugins declare their tenancy requirements explicitly. The block is the **contract**; table columns are the **implementation**. Core cross-checks the two and fails fast on inconsistency.

```ts
plugin('coupons', {
  support: {
    merchants: 'optional',
    branches: 'optional',
  },
  tables: [coupons],
})

plugin('business-hours', {
  support: {
    branches: 'required',
  },
  tables: [hours],
})

plugin('merchant-settings', {
  support: {
    merchants: 'required',
  },
  tables: [settings],
})

plugin('audit-log', {
  // no support block — tenancy-agnostic
  tables: [logs],
})
```

### Values

| Value | Meaning |
|---|---|
| `'required'` | Install fails if this tenancy feature isn't enabled |
| `'optional'` | Plugin works in both modes; behavior adapts |
| absent | Plugin doesn't interact with this dimension at all |

### Install-time validation

Core validates the active tenancy config against every installed plugin's `support` block during `createCommerce()`:

```
✗ Cannot install plugin 'business-hours':
  requires tenancy.branches = true, but it is disabled.
  Enable in createCommerce({ tenancy: { branches: true } }).

✗ Plugin 'coupons' declares support.merchants = 'optional'
  but table 'coupons' has merchant() as required.
  Either make the column optional or declare support.merchants = 'required'.
```

### Implied requirements

`support.branches: 'required'` transitively requires `support.merchants` to be enabled. Plugins don't have to declare both — core derives the implication.

## Plugin code is identical in any tenancy mode

The plugin author writes one version. Core adapts schema, writes, and reads.

```ts
export const couponsPlugin = plugin('coupons', {
  support: {
    merchants: 'optional',
    branches: 'optional',
  },
  tables: [coupons],

  operations: {
    'coupons:create': {
      handler: async ({ input }) => coupons.insert(input),
    },
    'coupons:findByCode': {
      handler: async ({ input }) => coupons.findUnique({ code: input.code }),
    },
  },

  on: {
    'orders:checkout:before': async ({ input, commerce }) => {
      if (input.couponCode) {
        // Hierarchical lookup — returns the most specific match for the current
        // merchant+branch, or the global coupon in simple-store mode.
        await commerce.coupons.findByCode({ code: input.couponCode })
      }
    },
  },
})
```

Customer journey (Jaicome shape):

| Actor | Creates coupon at level | Customer at branch L1 of merchant M1 sees |
|---|---|---|
| App admin | platform | yes |
| Merchant M1 owner | merchant M1 | yes |
| Merchant M2 owner | merchant M2 | no |
| L1 manager | branch L1 | yes |
| L2 manager | branch L2 (same M1) | no |

`findByCode` returns at most one row with explicit precedence: branch > merchant > platform.

## Write inference from request context

Tenancy column values come from the active `RequestContext` unless explicitly overridden:

```ts
// Merchant owner is logged in — context: { merchantId: 'm1' }
await commerce.coupons.create({ code: 'SUMMER25', discountValue: 1000 })
// → inserted with merchant = 'm1', branch = null

// Branch manager is logged in — context: { merchantId: 'm1', branchId: 'l1' }
await commerce.coupons.create({ code: 'STORE5', discountValue: 500 })
// → inserted with merchant = 'm1', branch = 'l1'

// Admin is logged in — no merchant/branch in context
await commerce.coupons.create({ code: 'BLACKFRIDAY', discountValue: 2000 })
// → inserted with merchant = null, branch = null (platform-wide)

// Admin explicitly creates on behalf of a merchant
await commerce.coupons.create({
  code: 'PIZZA10',
  discountValue: 1000,
  merchant: 'm1',
})
// → inserted with merchant = 'm1', branch = null
```

If the row requires a tenancy column and the context has no value and the input has no override, the write fails:

```
✗ Cannot insert into 'merchantSettings':
  column 'merchant' is required, but request context has no merchantId.
  Bind a context via commerce.withContext({ merchantId }) or pass merchant in input.
```

## Escape hatches

Three escape hatches cover the cases where automatic resolution isn't what you want.

### `acrossAll: true`

For admin UIs that need to see every row regardless of tenancy:

```ts
// Returns all coupons across all merchants and branches
await commerce.coupons.list({ acrossAll: true })
```

Authorization for `acrossAll: true` is the caller's responsibility — the SDK does not restrict it.

### `where: { merchant: null }`

Filter explicitly by tenancy column value, including null for platform-only:

```ts
// Only platform-level coupons, no merchant overrides
await commerce.coupons.list({ where: { merchant: null } })

// Only one specific merchant's coupons
await commerce.coupons.list({ where: { merchant: 'm1' } })
```

### Explicit override in input

Pass the tenancy column directly to bypass context inference:

```ts
await commerce.coupons.create({
  code: 'X',
  merchant: 'm1',
  branch: 'l1',
})
```

## Built-in tenancy entities

When `tenancy.merchants: true`, core materializes a `merchants` table:

```ts
merchant = {
  id: string
  name: string
  slug: string
  status: 'active' | 'suspended' | 'archived'
  metadata: Record<string, unknown>
  createdAt: Date
  updatedAt: Date
}
```

When `tenancy.branches: true`, core also materializes a `branches` table:

```ts
branch = {
  id: string
  merchantId: string             // every branch belongs to a merchant
  name: string
  slug: string
  address: Address               // required — drives delivery and fulfillment routing
  coordinates?: Coordinates      // required by adapters that need geolocation
  timezone: string               // IANA TZ identifier
  status: 'active' | 'archived'
  metadata: Record<string, unknown>
  createdAt: Date
  updatedAt: Date
}
```

The corresponding `commerce.merchants.*` and `commerce.branches.*` SDK namespaces appear on the typed SDK only when their tenancy axis is enabled.

## Core entity participation

Core entities also participate in tenancy when enabled. Their declarations live alongside the entity definitions in [20-data-model.md](./20-data-model.md), but the rules are the same:

| Entity | `merchant()` | `branch()` |
|---|---|---|
| `product` | optional | optional |
| `productVariant` | inherits from product | inherits from product |
| `order` | required (when merchants on) | optional |
| `customer` | optional | absent |
| `category` | optional | optional |

`order.merchant` is required because every order in a multi-merchant deployment must attribute to exactly one merchant (orders cannot span merchants — that's the marketplace checkout mode boundary, see [45-marketplace-mode.md](./45-marketplace-mode.md)).

## Cross-links

- Plugin authoring surface, `plugin()` and `table()` factories: [40-plugin-system.md](./40-plugin-system.md)
- Marketplace UX layered on `tenancy.merchants`: [45-marketplace-mode.md](./45-marketplace-mode.md)
- `RequestContext` shape and `withContext` binding: [25-server-sdk.md](./25-server-sdk.md)
- Type inference from `createCommerce()`: [60-type-inference.md](./60-type-inference.md)

## Future RFCs

- Per-tenant adapter configuration (different payment providers per merchant)
- Tenant-scoped feature flags
- Tenancy axes beyond merchants and branches (e.g., currency-scoped, region-scoped)
