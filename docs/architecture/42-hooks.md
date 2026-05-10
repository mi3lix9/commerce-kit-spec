# Hooks

## Purpose

Define the app-level hook system for Commerce Kit: how developers register `before` and `after` middleware at `createCommerce()` time, the context shape each hook receives, type narrowing by operation, and the `runInBackground` utility for post-commit side effects.

Plugin-level hooks follow the same `before`/`after` model but use a `next()` chain because multiple plugins compose in sequence. That contract is defined in [40-plugin-system.md](./40-plugin-system.md). This document defines the app-level surface and the shared context shapes both layers use.

## Non-goals

- Plugin-level hook chaining and `next()` semantics — see [40-plugin-system.md](./40-plugin-system.md)
- Incoming webhook verification from payment or fulfillment providers — see [80-framework-adapters.md](./80-framework-adapters.md)
- Durable background job queues or outbox patterns — out of scope for v1

## Core decisions

- App-level hooks are registered at `createCommerce()` time via a `hooks` config key.
- Hooks are `before` and `after` — not a single `next()`-based middleware chain.
- `before` hooks run before plugin before-hooks and the operation. `after` hooks run after the operation and plugin after-hooks.
- Checking `ctx.operation` narrows `ctx.input` and `ctx.result` to the correct types for that operation.
- `after` hooks provide `ctx.runInBackground()` for scheduling post-commit work that does not block the response.
- App-level hooks and plugin-level hooks use the same `createHook` factory and the same context shapes.

## Registration

```ts
import { createCommerce, createHook } from 'commerce-kit'

const commerce = createCommerce({
  database: drizzleAdapter({ db }),
  payments: [moyasarPayments(...)],

  hooks: {
    before: createHook(async (ctx) => {
      if (ctx.operation === 'orders:checkout') {
        if (isBlockedCustomer(ctx.request.customerId)) {
          throw new CommerceError('Account suspended')
        }
      }
    }),

    after: createHook(async (ctx) => {
      if (ctx.operation === 'orders:checkout') {
        ctx.runInBackground(async () => {
          await sendOrderConfirmationEmail(ctx.result.order)
        })
      }
    }),
  },

  onBackgroundError: (error, ctx) => {
    logger.error('Background hook failed', { operation: ctx.operation, error })
  },
})
```

## `before` hook

### Context shape

```ts
type BeforeHookContext = {
  operation: OperationKey           // e.g. 'orders:checkout'
  input: OperationInput[operation]  // typed per operation after narrowing
  request: RequestContext           // customerId, actorId, roles, locale, metadata
}
```

### Behavior

| Return value       | Effect                                              |
|--------------------|-----------------------------------------------------|
| `void` / `undefined` | Proceed with the original input                   |
| Modified input object | Replace the operation's input with the returned value |
| throw              | Abort the operation; the error is returned to the caller |

Returning a modified input is the hook-level equivalent of `transformFunctionInput` — it lets the before hook enrich or sanitize input before the operation and plugins run.

```ts
before: createHook(async (ctx) => {
  if (ctx.operation === 'orders:checkout') {
    // annotate every checkout with the request source
    return {
      ...ctx.input,
      metadata: { ...ctx.input.metadata, source: 'web' },
    }
  }
})
```

## `after` hook

### Context shape

```ts
type AfterHookContext = {
  operation: OperationKey             // e.g. 'orders:checkout'
  input: OperationInput[operation]    // the original input
  result: OperationResult[operation]  // typed per operation after narrowing
  request: RequestContext
  runInBackground: (fn: () => Promise<void>) => void
}
```

### Behavior

| Return value         | Effect                                              |
|----------------------|-----------------------------------------------------|
| `void` / `undefined` | Pass the original result through to the caller      |
| Modified result object | Replace the operation's result with the returned value |
| throw                | The operation fails; the error is returned to the caller |

### `runInBackground`

`runInBackground` schedules a function to run after the database transaction commits. It does not block the HTTP response. Errors thrown inside it are captured and forwarded to `onBackgroundError`; they do not affect the operation's HTTP response status.

```ts
after: createHook(async (ctx) => {
  if (ctx.operation === 'orders:checkout') {
    // does not delay the checkout response
    ctx.runInBackground(async () => {
      await sendOrderConfirmationEmail(ctx.result.order)
      await analytics.track('order_placed', { orderId: ctx.result.order.id })
    })
  }

  if (ctx.operation === 'orders:cancelled') {
    ctx.runInBackground(async () => {
      await notifyFulfillmentTeam(ctx.result.order)
    })
  }
})
```

Work registered via `runInBackground` is fire-and-forget from Commerce Kit's perspective. If guaranteed delivery is required, the background function should enqueue a job using the application's own job queue rather than performing the side effect directly.

## Type narrowing by operation

Checking `ctx.operation` narrows both `ctx.input` and `ctx.result` to the correct types. No casting is required.

```ts
after: createHook(async (ctx) => {
  if (ctx.operation === 'orders:checkout') {
    ctx.result   // → CheckoutResult: { order: Order; paymentReference: string }
    ctx.input    // → CheckoutInput
  }

  if (ctx.operation === 'products:create') {
    ctx.result   // → Product
    ctx.input    // → CreateProductInput
  }
})
```

The full `OperationKey` union is derived from the installed plugin and adapter set at `createCommerce()` time. Plugin-contributed operations become valid `ctx.operation` values after narrowing when the plugin is installed.

## Execution order

For each operation:

1. App-level `before` hook
2. Plugin `before` hooks in declaration order
3. Operation handler (inside the database transaction)
4. Plugin `after` hooks in declaration order
5. App-level `after` hook
6. Transaction commits
7. `runInBackground` callbacks execute post-commit

The app-level hook wraps the full plugin and operation stack. Plugin after-hooks complete before the app-level after hook runs, so the after hook sees the result after any plugin-level post-processing.

## Operation keys

These are the valid `ctx.operation` values from core. Plugin-contributed operations extend this set when the plugin is installed.

### Always available

```
products:create       products:update       products:archive
products:getVersion
orders:checkout       orders:cancel         orders:confirm
orders:setPaid        orders:refund
customers:create      customers:update
categories:create     categories:update     categories:archive
```

### Adapter-gated (only when configured)

```
fulfillment:createMethod    fulfillment:updateMethod    fulfillment:archiveMethod
fulfillment:createFulfillment
```

## Plugin-level hooks

Plugins use the same `createHook` factory and receive the same context shapes. The difference is that plugin before-hooks must call `next()` or throw, because multiple plugins compose into a chain.

```ts
const auditPlugin = definePlugin({
  id: 'audit-log',
  hooks: {
    after: createHook(async (ctx) => {
      await db.insert(auditLog).values({
        operation: ctx.operation,
        actorId: ctx.request.actorId ?? null,
        at: new Date(),
      })
    }),
  },
})
```

Plugin hooks that need post-commit side effects use the same `ctx.runInBackground` utility.

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

- Plugin hook chaining and `next()` contract: [40-plugin-system.md](./40-plugin-system.md)
- `RequestContext` shape and `resolveContext`: [80-framework-adapters.md](./80-framework-adapters.md)
- Type inference for plugin-contributed operations: [60-type-inference.md](./60-type-inference.md)

## Future RFCs

- Additional operation keys from post-v1 core domains
- Ordered `runInBackground` with dependency between background callbacks
- Retry policy for background callbacks
