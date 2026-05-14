# Hooks

## Purpose

Define the keyed `on` handler model used at both app-level (via `createCommerce()`) and plugin-level. Handlers are keyed by **operation:phase**, automatically type-narrowed, and support wildcards for cross-cutting concerns.

This document defines the shared handler shape, context fields, transaction semantics, and `runInBackground` behavior. The plugin-side surface is in [40-plugin-system.md](./40-plugin-system.md).

## Non-goals

- Incoming webhook verification — see [80-framework-adapters.md](./80-framework-adapters.md)
- Durable background job queues — see [43-background-tasks.md](./43-background-tasks.md)
- Tenancy-aware scoping (`merchant()`, `branch()` resolution) — see [12-tenancy.md](./12-tenancy.md)

## Core decisions

- Handlers are registered under an `on` map keyed by `operation:phase` (e.g., `'orders:checkout:before'`).
- The key narrows `input`/`result` types automatically — no `if (ctx.operation === ...)` ladders.
- Wildcards (`*:after`, `orders:*`, `*`) are supported for cross-cutting plugins.
- Handler arguments are **destructured**, not pulled off a `ctx` object.
- `:before` handlers return `void` to proceed, return a modified input to transform, or throw to abort.
- `:after` handlers return `void` to pass through, return a modified result to transform, or throw to fail.
- `:after` handlers provide `runInBackground` for post-commit side effects.
- App-level handlers wrap plugin-level handlers in execution order.

## Registration

```ts
import { createCommerce } from 'commerce-kit'

const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payments: [moyasar({ secretKey: env.MOYASAR_SECRET })],

  on: {
    'orders:checkout:before': async ({ input, request }) => {
      if (isBlockedCustomer(request.customerId)) {
        throw new CommerceError('Account suspended')
      }
    },

    'orders:checkout:after': async ({ result, runInBackground }) => {
      runInBackground(async () => {
        await sendOrderConfirmationEmail(result.order)
      })
    },
  },

  onBackgroundError: (error, ctx) => {
    logger.error('Background handler failed', { operation: ctx.operation, error })
  },
})
```

## Handler key syntax

```
<namespace>:<method>:<phase>
```

| Pattern | Matches |
|---|---|
| `'orders:checkout:before'` | exact phase of exact operation |
| `'orders:checkout:after'` | exact phase of exact operation |
| `'orders:checkout:*'` | both phases of one operation |
| `'orders:*'` | every phase of every `orders:` operation |
| `'*:before'` | before phase of every operation |
| `'*:after'` | after phase of every operation |
| `'*'` | every phase of every operation |

Wildcards make cross-cutting handlers (audit log, analytics, request tracing) ergonomic.

## `:before` handler

### Argument shape

```ts
{
  operation: OperationKey     // narrowed to the matching key
  input: OperationInput       // narrowed per operation — no manual checks
  request: RequestContext     // customerId, actorId, merchantId, branchId, etc.
  commerce: CommerceSDK       // pre-scoped to current request context
}
```

### Behavior

| Return value | Effect |
|---|---|
| `void` / `undefined` | Proceed with the original input |
| Modified input object | Replace the operation's input with the returned value |
| throw | Abort the operation; error returned to the caller |

```ts
'orders:checkout:before': async ({ input }) => {
  // annotate every checkout with the request source
  return {
    ...input,
    metadata: { ...input.metadata, source: 'web' },
  }
}
```

## `:after` handler

### Argument shape

```ts
{
  operation: OperationKey
  input: OperationInput
  result: OperationResult     // narrowed per operation
  request: RequestContext
  commerce: CommerceSDK
  runInBackground: (fn: () => Promise<void>) => void
}
```

### Behavior

| Return value | Effect |
|---|---|
| `void` / `undefined` | Pass the original result through |
| Modified result object | Replace the operation's result with the returned value |
| throw | The operation fails; error returned to the caller |

### `runInBackground`

`runInBackground` schedules a function to run **after the database transaction commits**. It does not block the HTTP response. Errors thrown inside the callback are forwarded to `onBackgroundError`; they do not affect the response status.

```ts
'orders:checkout:after': async ({ result, runInBackground }) => {
  runInBackground(async () => {
    await sendOrderConfirmationEmail(result.order)
    await analytics.track('order_placed', { orderId: result.order.id })
  })
}
```

Work registered via `runInBackground` is fire-and-forget from Commerce Kit's perspective. If guaranteed delivery is required, the background function should enqueue a durable job via the application's job queue rather than performing the side effect directly.

## Type narrowing

The key narrows both `input` and `result` automatically:

```ts
on: {
  'orders:checkout:after': async ({ result }) => {
    result.order             // → Order
    result.paymentReference  // → string
    result.paymentUrl        // → string | undefined
  },

  'products:create:after': async ({ result }) => {
    result                   // → Product
  },
}
```

The full `OperationKey` union is derived from the installed plugin and adapter set at `createCommerce()` time. Plugin-contributed operations become valid keys when the plugin is installed.

### Narrowing under wildcards

Wildcard handlers receive widened types. The handler is responsible for narrowing manually if it needs to:

```ts
on: {
  '*:after': async ({ operation, result }) => {
    if (operation === 'orders:checkout') {
      result.order  // narrowed
    }
  },
}
```

## Execution order

For each operation:

1. App-level `:before` handlers in declaration order
2. Plugin `:before` handlers in plugin declaration order
3. Operation handler (inside the database transaction)
4. Plugin `:after` handlers in plugin declaration order
5. App-level `:after` handlers in declaration order
6. Transaction commits
7. `runInBackground` callbacks execute post-commit

App-level handlers wrap the full plugin and operation stack: app-before runs first, app-after runs last. Plugin `:after` handlers complete before the app-level `:after` handler runs, so the app handler sees the result after any plugin-level post-processing.

Wildcards within a single source (app or plugin) run alongside exact matches in declaration order — no priority between specific and wildcard handlers.

## Transaction boundary

- `:after` means after the core handler logic, **not** after database commit.
- Commerce Kit must not commit until all participating `:after` handlers complete successfully.
- Throwing inside `:after` rolls back the transaction.
- For post-commit side effects (sending notifications, calling external services), use `runInBackground`.

## `request` context

```ts
type RequestContext = {
  customerId?: string | null
  actorId?: string | null
  actorType?: string          // 'customer' | 'merchant' | 'admin' | ...
  roles?: string[]
  permissions?: string[]
  merchantId?: string         // present when tenancy.merchants is on
  branchId?: string         // present when tenancy.branches is on
  locale?: string
  metadata?: Record<string, unknown>
}
```

Tenancy fields (`merchantId`, `branchId`) drive automatic write inference and hierarchical reads for tables that declare `merchant()` / `branch()` columns. See [12-tenancy.md](./12-tenancy.md).

## Operation keys

Core operation keys (always available):

```
products:create        products:update        products:archive
orders:checkout        orders:cancel          orders:confirm
orders:setPaid         orders:refund
customers:create       customers:update
categories:create      categories:update      categories:archive
```

Tenancy-gated (only when configured):

```
merchants:create       merchants:update       merchants:archive
branches:create       branches:update       branches:archive
```

Adapter-gated (only when configured):

```
fulfillment:createMethod    fulfillment:updateMethod    fulfillment:archiveMethod
fulfillment:createFulfillment
```

Plugin-contributed operations extend this set when the plugin is installed.

## State-change events

In addition to `operation:phase` hooks, the order state machine emits **state-change events** whenever an order transitions into a terminal or notable state, regardless of whether the transition originated from a typed operation, a payment webhook, or a plugin-driven flow. These events are intentionally trigger-agnostic so that downstream concerns (auto-dispatch, notifications, ledger projections) react to the **state**, not the **operation** that produced it.

```
orders:confirmed       orders:fulfilled       orders:cancelled
orders:completed       orders:refunded
```

Format: `<resource>:<newState>` (no `:before` / `:after` phase — state changes have already happened by the time the event fires; treat them as `after`-style notifications).

Example — auto-dispatch listens to `orders:confirmed`, not `orders:confirm:after`, so it also fires when the order is confirmed via a payment webhook:

```ts
on: {
  'orders:confirmed': async ({ order, commerce, runInBackground }) => {
    if (order.deliveryMethodId) {
      runInBackground(() => commerce.delivery.create({ orderId: order.id }))
    }
  },
}
```

A small number of subsystems also expose **resolver hooks** that follow neither format — e.g., `'calculation:resolvePipeline'` in [35-calculation-engine.md](./35-calculation-engine.md). These are documented at their source.

## `onBackgroundError`

Errors thrown inside `runInBackground` callbacks are forwarded to `onBackgroundError`. If not configured, errors are silently discarded.

```ts
createCommerce({
  onBackgroundError: (error, ctx) => {
    captureException(error, { extra: { operation: ctx.operation } })
  },
})
```

## Cross-links

- Plugin authoring surface and `plugin()` factory: [40-plugin-system.md](./40-plugin-system.md)
- Tenancy column helpers and hierarchical reads: [12-tenancy.md](./12-tenancy.md)
- `RequestContext` resolution from HTTP requests: [80-framework-adapters.md](./80-framework-adapters.md)
- Type inference for plugin-contributed operations: [60-type-inference.md](./60-type-inference.md)

## Future RFCs

- Ordered `runInBackground` with dependency between background callbacks
- Retry policy for background callbacks
- Wildcard priority resolution if multiple plugins need ordered cross-cutting concerns
