# Plugin System

## Purpose

Define the Commerce Kit plugin authoring surface: the `plugin()` factory, `table()` column builders, `support` block, `operations`, keyed `on` handlers, tasks, webhooks, and tenancy-agnostic data access.

The plugin system is designed so that **plugins are tenancy-agnostic by default**. The same plugin code runs in simple-store, multi-merchant, and multi-branch modes — core handles schema generation, write inference, and hierarchical reads.

## Non-goals

- Tenancy semantics (column helpers, hierarchical resolution, install validation) — see [12-tenancy.md](./12-tenancy.md)
- Core entity persistence rules — see [20-data-model.md](./20-data-model.md)
- Hook context shapes and execution order — see [42-hooks.md](./42-hooks.md)
- Adapter contracts — see [50-adapter-system.md](./50-adapter-system.md)
- Schema validation contract for `operations.input` — see [47-validation.md](./47-validation.md)

## Core decisions

- Plugins use a single `plugin('id', config)` factory. No class hierarchy, no `definePlugin` wrapper.
- Tables are first-class values declared with `table('name', columns)`. The table value carries its own typed methods (`findUnique`, `insert`, etc.).
- Tenancy is declared explicitly through a `support` block. Mismatches with the active `createCommerce()` tenancy config fail at install time.
- Hooks are **keyed by operation and phase** (`'orders:checkout:before'`), not a discriminated-union ladder.
- Operations the plugin contributes to `commerce.*` are declared in a single `operations` block.
- Plugin code is identical across all tenancy modes — column helpers (`merchant()`, `branch()`) handle the difference declaratively.

## `plugin()` factory

```ts
import { plugin, table, text, integer } from 'commerce-kit'

export const couponsPlugin = plugin('coupons', {
  version: '0.1.0',

  support: {
    merchants: 'optional',
    branches: 'optional',
  },

  tables: [coupons],

  operations: {
    'coupons:create': {
      input: CreateCouponInput,
      handler: async ({ input }) => coupons.insert(input),
    },
    'coupons:findByCode': {
      input: FindByCodeInput,
      handler: async ({ input }) => coupons.findUnique({ code: input.code }),
    },
  },

  on: {
    'orders:checkout:before': async ({ input, commerce }) => {
      if (input.couponCode) {
        await commerce.coupons.findByCode({ code: input.couponCode })
      }
    },
  },

  tasks: {
    'coupons:expire': async () => {
      // expire all coupons whose endDate is in the past
    },
  },
})
```

### Plugin contract

```ts
type Plugin = {
  id: string                        // first positional arg to plugin()
  version: string                   // semver, matches package version

  support?: TenancySupport          // declared tenancy requirements
  requires?: RuntimeRequirement[]   // plugin or adapter dependencies
  tables?: Table[]                  // schema contributions

  operations?: PluginOperations     // typed namespace additions to commerce.*
  on?: PluginHandlers               // keyed lifecycle handlers
  calculation?: PluginCalculation   // named pricing pipeline steps
  fulfillment?: PluginFulfillment   // fulfillment type contributions

  tasks?: PluginTasks               // named tasks for commerce.tasks.run
  webhooks?: PluginWebhooks         // verified-event handlers
  states?: PluginStates             // order state machine extensions

  onInit?: (ctx: InitContext) => Promise<void>
  onTeardown?: (ctx: InitContext) => Promise<void>
}
```

### `id` and `version`

- `id` is unique, kebab-case, used in error messages, task keys, operation keys, and the conflict-detection registry.
- `version` is semver and must match the published npm package version.

### Tenancy support

See [12-tenancy.md](./12-tenancy.md) for the full `support` contract.

```ts
type TenancySupport = {
  merchants?: 'required' | 'optional'
  branches?: 'required' | 'optional'
}
```

| Value | Meaning |
|---|---|
| `'required'` | Install fails if this tenancy feature isn't enabled |
| `'optional'` | Plugin works in both modes; behavior adapts |
| absent | Plugin doesn't interact with this dimension |

Core validates `support` against the active tenancy config and against every declared table's column declarations during `createCommerce()`.

## `table()` and column helpers

Tables are first-class values that carry their schema and a typed data-access surface.

```ts
import { table, text, integer, jsonb, enumOf, merchant, branch } from 'commerce-kit'

const coupons = table('coupons', {
  code: text().unique(),
  discountType: enumOf(['percentage', 'fixed']),
  discountValue: integer(),
  startsAt: timestamp().nullable(),
  endsAt: timestamp().nullable(),
  merchant: merchant().optional(),
  branch: branch().optional(),
})

// At runtime, the table value exposes typed methods:
coupons.insert({ code: 'SUMMER25', discountType: 'percentage', discountValue: 25 })
coupons.findUnique({ code: 'SUMMER25' })
coupons.findFirst({ where: { discountValue: { gte: 1000 } } })
coupons.findMany({ where: { ... }, orderBy: ..., limit: 20 })
coupons.update({ where: { id: 'cpn_1' }, data: { discountValue: 1500 } })
coupons.delete({ where: { id: 'cpn_1' } })
```

### Column helpers

Plain column types: `text()`, `integer()`, `bigint()`, `boolean()`, `timestamp()`, `jsonb()`, `enumOf([...])`, `decimal()`.

Modifiers: `.unique()`, `.nullable()`, `.default(value)`, `.references('tableName')`.

Tenancy helpers: `merchant()`, `branch()`. Their semantics are defined in [12-tenancy.md](./12-tenancy.md).

### Scoping is implicit

When a table declares `merchant()` and/or `branch()`, all reads and writes on that table are automatically scoped through the current request context. The plugin author writes `coupons.findUnique({ code })` and gets the right hierarchical result without thinking about scope.

## `operations`

`operations` adds typed namespaces to `commerce.*`. Each operation declares its input schema and handler. The operation key follows `namespace:method` and must be unique across the system.

The `input` field accepts any Standard Schema-compatible value (Zod, Valibot, Arktype, Effect Schema, etc.). Examples in this document use Zod for readability. See [47-validation.md](./47-validation.md) for the validation contract.

```ts
operations: {
  'coupons:create': {
    input: z.object({ code: z.string(), discountValue: z.number() }),
    handler: async ({ input, commerce }) => coupons.insert(input),
  },
  'coupons:validate': {
    input: z.object({ code: z.string() }),
    handler: async ({ input }) => {
      const c = await coupons.findUnique({ code: input.code })
      if (!c) throw new CommerceError('Invalid coupon')
      return c
    },
  },
}
```

After install, these become typed methods on `commerce.*`:

```ts
await commerce.coupons.create({ code: 'X', discountValue: 1000 })
await commerce.coupons.validate({ code: 'X' })
```

Operation keys are also valid `on:` handler keys — other plugins can hook into them.

### Handler argument

Operation handlers receive a single destructured argument:

```ts
{
  input: TypedInput          // narrowed from the operation's input schema
  commerce: CommerceSDK      // pre-scoped to the current request context
  request: RequestContext    // customerId, actorId, merchantId, branchId, etc.
}
```

`commerce` is the fully-typed SDK including all installed plugins. Operation handlers can call other operations directly without manual context-passing.

### Collision policy

- Operation keys must be unique across core + all installed plugins.
- Namespace prefixes (e.g., `coupons:` in `coupons:create`) must be unique per plugin id by convention.
- Collisions fail at `createCommerce()` initialization with a concrete error.

## `on` handlers

Lifecycle handlers are keyed by **operation:phase**. Each key maps to a typed handler.

```ts
on: {
  'orders:checkout:before': async ({ input, request }) => {
    // input and request are narrowed automatically — no `if (ctx.operation === ...)` needed
    return { ...input, metadata: { ...input.metadata, source: 'web' } }
  },

  'orders:checkout:after': async ({ result, runInBackground }) => {
    runInBackground(async () => {
      await sendOrderConfirmationEmail(result.order)
    })
  },

  // Wildcards
  '*:after': async ({ operation, result }) => {
    // runs after every operation — useful for audit logs and analytics
  },

  'orders:*': async ({ operation, input, result }) => {
    // runs for every orders operation, before and after
  },
}
```

### Handler argument

Each handler receives a single destructured argument. The keys depend on the phase:

**`:before` handlers:**

```ts
{
  operation: OperationKey   // narrowed to the matching key
  input: TypedInput         // narrowed per operation
  request: RequestContext
  commerce: CommerceSDK
}
```

Return `void` to proceed, return a modified input object to transform, throw to abort.

**`:after` handlers:**

```ts
{
  operation: OperationKey
  input: TypedInput
  result: TypedResult       // narrowed per operation
  request: RequestContext
  commerce: CommerceSDK
  runInBackground: (fn: () => Promise<void>) => void
}
```

Return `void` to pass result through, return a modified result to transform, throw to fail.

See [42-hooks.md](./42-hooks.md) for the full hook context, type narrowing rules, transaction semantics, and `runInBackground` behavior.

### Wildcards

| Pattern | Matches |
|---|---|
| `'orders:checkout:before'` | exact phase of exact operation |
| `'orders:checkout:*'` | both phases of exact operation |
| `'orders:*'` | every phase of every `orders:` operation |
| `'*:before'` | before phase of every operation |
| `'*:after'` | after phase of every operation |
| `'*'` | every phase of every operation |

Wildcards make cross-cutting plugins (audit log, analytics, request tracing) ergonomic without manual switching.

### Execution order

For each operation:

1. App-level `before` handlers (registered on `createCommerce()`)
2. Plugin `before` handlers in declaration order
3. Operation handler (inside the database transaction)
4. Plugin `after` handlers in declaration order
5. App-level `after` handlers
6. Transaction commits
7. `runInBackground` callbacks execute post-commit

Wildcards run alongside exact matches in declaration order — there is no priority between specific and wildcard handlers within a plugin.

## `fulfillments`

Plugins contribute fulfillment types to core's open type registry. Each type declares a typed `data` schema validated when orders reference the type. Full semantics in [52-fulfillment-types.md](./52-fulfillment-types.md).

```ts
plugin('restaurant', {
  fulfillments: [
    fulfillmentType('restaurant:dinein', {
      data: z.object({ tableNumber: z.string() }),
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

Type IDs must use the `<plugin-id>:<name>` convention. Core pre-registers `'shipping'`, `'delivery'`, `'pickup'`, `'digital'` with their own schemas; plugin types are equal citizens in the registry.

## `calculation`

Plugins contribute named calculation steps that core's pricing pipeline can reference by ID. Step authoring rules and the full `StepContext` shape live in [35-calculation-engine.md](./35-calculation-engine.md).

```ts
plugin('pricing-rules', {
  calculation: {
    steps: {
      'pricing-rules:discounts': {
        type: 'discount',
        handler: async ({ snapshot, emit, commerce, request }) => { /* ... */ },
      },
      'pricing-rules:taxes': {
        type: 'tax',
        handler: async ({ snapshot, emit }) => { /* ... */ },
      },
    },
  },
})
```

Step IDs must be unique across the installed plugin set. The convention is `<plugin-id>:<name>`. Pipelines (in app config or stored per-merchant) reference these IDs as plain strings; the engine resolves them through the registry at calculation time.

## `tasks`

Plugins contribute named tasks invocable via `commerce.tasks.run(key)`. Task keys must be namespaced with the plugin id.

```ts
tasks: {
  'coupons:expire': async ({ commerce, request }) => {
    // expire all coupons whose endDate is in the past
  },
}

// available after install:
await commerce.tasks.run('coupons:expire')
```

The task handler argument is `{ commerce, request, options }` where `options` is the optional second argument to `commerce.tasks.run(key, options)`.

See [43-background-tasks.md](./43-background-tasks.md) for the full tasks API and scheduling.

## `webhooks`

Plugins consume verified webhook events. Adapters perform signature verification and parsing; plugins receive only the normalized event.

```ts
webhooks: {
  'payment.captured': async ({ event, commerce }) => {
    await commerce.orders.setPaid({ id: event.orderId })
  },
}
```

### Webhook rules

- Keys are normalized verified event type strings produced by the active adapter contract after `verifyWebhook(...)` succeeds.
- A plugin must not assume provider-native event names if the adapter contract normalizes them.
- Dispatch is sequential in plugin declaration order. If multiple plugins subscribe to the same event type, each matching handler runs.
- If one handler fails, dispatch to later handlers for that event stops and the webhook is treated as unsuccessfully processed.
- A webhook request returns HTTP 200 only after adapter verification, core event handling, and plugin dispatch all succeed.
- Retry ownership remains with the upstream provider.

## `states`

Plugins may extend the order state machine with additional states and transitions. This is the only valid extension point for state-machine mutation.

```ts
states: {
  'awaiting_review': {
    from: ['placed'],
    to: ['confirmed', 'cancelled'],
  },
}
```

Rules:

- Base states remain defined by the core data model ([20-data-model.md](./20-data-model.md)).
- All transitions must be declared. Undeclared transitions fail at runtime.
- Plugins may not redefine base state meanings; they only add states and transitions.

## `requires` — runtime dependencies

```ts
type RuntimeRequirement =
  | { type: 'plugin'; pluginId: string; version?: string }
  | { type: 'config'; key: string; value?: unknown; optional?: boolean }
  | { type: 'adapter'; adapter: 'payment' | 'storage' | 'fulfillment' | 'payout' | 'scheduler' }
```

Validated during `createCommerce()`. Use `requires[]` for runtime topology and config guarantees, not for package installation — that belongs in `peerDependencies`.

## Plugin-owned data access

Plugins own their data through `tables`. Reads and writes happen through table methods, not raw database clients.

- Plugin reads and writes happen through Commerce Kit's database adapter, never through out-of-band clients.
- When invoked inside a core transactional operation, plugin reads/writes and both `:before`/`:after` handlers participate in that same transaction.
- Plugins must not assume a separate private transaction boundary unless they explicitly open a nested adapter transaction where the adapter supports it.

## Transaction boundary rule

- `:after` means after the core handler logic, not after database commit.
- Commerce Kit must not commit a transactional operation until all participating `:after` handlers complete successfully.
- Plugin authors must treat `:after` handlers as part of the same atomic operation when a transaction exists.
- If a plugin needs post-commit or best-effort side effects, use `runInBackground` rather than performing side effects directly.

## Forbidden behavior

Plugins must not:

- Import another plugin's internal functions directly. Cross-plugin interaction goes through `commerce.*` operations.
- Issue raw database queries that bypass the table abstraction (and therefore bypass tenancy scoping).
- Register a conflicting plugin `id`, operation key, or table name.
- Mutate the order state machine outside the `states` declaration.
- Replace adapter webhook signature verification.

## Worked example — a plugin that supports any tenancy mode

```ts
import {
  plugin,
  table,
  text,
  integer,
  enumOf,
  timestamp,
  merchant,
  branch,
} from 'commerce-kit'
import { z } from 'zod'

const coupons = table('coupons', {
  code: text().unique(),
  discountType: enumOf(['percentage', 'fixed']),
  discountValue: integer(),
  endsAt: timestamp().nullable(),
  merchant: merchant().optional(),
  branch: branch().optional(),
})

export const couponsPlugin = plugin('coupons', {
  version: '0.1.0',

  support: {
    merchants: 'optional',
    branches: 'optional',
  },

  tables: [coupons],

  operations: {
    'coupons:create': {
      input: z.object({
        code: z.string(),
        discountType: z.enum(['percentage', 'fixed']),
        discountValue: z.number().int().positive(),
        endsAt: z.date().nullable().optional(),
      }),
      handler: async ({ input }) => coupons.insert(input),
    },

    'coupons:findByCode': {
      input: z.object({ code: z.string() }),
      handler: async ({ input }) => coupons.findUnique({ code: input.code }),
    },
  },

  on: {
    'orders:checkout:before': async ({ input, commerce }) => {
      if (input.couponCode) {
        const coupon = await commerce.coupons.findByCode({ code: input.couponCode })
        if (!coupon) throw new CommerceError('Invalid coupon')
        if (coupon.endsAt && coupon.endsAt < new Date()) {
          throw new CommerceError('Coupon expired')
        }
      }
    },
  },

  tasks: {
    'coupons:expire': async () => {
      // backend implementation
    },
  },
})
```

This single plugin works in all four configurations:

| `createCommerce` config | Coupon shape | Customer sees |
|---|---|---|
| no tenancy | one global table | any active coupon |
| `merchants: true` | platform + per-merchant | most specific match |
| `merchants: true, branches: true` | platform + per-merchant + per-branch | most specific match |

No branching in plugin code. No two-version maintenance. Tenancy is invisible until the plugin author specifically wants to query across scopes.

## Cross-links

- Tenancy contract (`merchant()`, `branch()`, `support`, hierarchical reads): [12-tenancy.md](./12-tenancy.md)
- Hook context, type narrowing, `runInBackground`: [42-hooks.md](./42-hooks.md)
- Background tasks API: [43-background-tasks.md](./43-background-tasks.md)
- Adapter contracts: [50-adapter-system.md](./50-adapter-system.md)
- Type inference from `createCommerce()`: [60-type-inference.md](./60-type-inference.md)
- Marketplace as a tenancy mode (cross-merchant checkout): [45-marketplace-mode.md](./45-marketplace-mode.md)

## Future RFCs

- Additional plugin capability surfaces beyond the current contract
- Inter-plugin event bus (separate from operation hooks)
- Plugin discoverability / registry contract
