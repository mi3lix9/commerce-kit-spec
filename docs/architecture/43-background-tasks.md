# Background Tasks

## Purpose

Define how Commerce Kit handles work that runs outside the request lifecycle: immediate post-commit side effects, deferred tasks triggered by commerce events, and recurring maintenance tasks invoked on a schedule.

## Non-goals

- Defining a built-in job runner or process scheduler — Commerce Kit does not own execution infrastructure
- Repeating hook semantics — `runInBackground` is defined in [42-hooks.md](./42-hooks.md)
- Outgoing webhook delivery queues — out of scope for v1

## Core decisions

- Background work is split into three layers: immediate (`runInBackground`), deferred (scheduler adapter), and recurring (`commerce.tasks`).
- Commerce Kit owns the commerce logic for deferred and recurring tasks; the execution infrastructure is supplied by the application through an optional scheduler adapter.
- The scheduler adapter follows the same dormant activation pattern as fulfillment and payout adapters.
- All schedulable work — from core and from plugins — is accessible through the `commerce.tasks` namespace.
- `commerce.tasks.run(taskKey)` is a plain async function; it does not require a scheduler adapter to be configured.

## Three layers

### Layer 1 — Immediate: `runInBackground`

Post-commit fire-and-forget work registered inside an `after` hook. No persistence, no retry. Suitable for sending emails, pinging analytics, or calling a webhook.

Defined in [42-hooks.md](./42-hooks.md).

### Layer 2 — Deferred: scheduler adapter

Work triggered by a commerce event but intentionally delayed. Requires persistence and retry. Activated only when a scheduler adapter is configured.

Examples:
- Cancel an order if not confirmed within 30 minutes of placement
- Poll the payment provider if no payment webhook arrives within 10 minutes
- Poll the fulfillment provider if no status update arrives within 1 hour

### Layer 3 — Recurring: `commerce.tasks`

Named maintenance operations designed to be called on a schedule from external cron infrastructure. No adapter required — these are plain async functions.

Examples:
- Expire abandoned orders nightly
- Expire stale coupons at their end date
- Sync fulfillment statuses hourly

---

## Scheduler adapter

### Activation

The scheduler adapter is optional. When not configured, deferred task features are dormant: no schema, no queuing, no task scheduling happens automatically.

```ts
import { bullmq } from '@commerce-kit/bullmq'

const commerce = createCommerce({
  database: drizzleAdapter(db, { schema }),
  payment: {
    moyasar: moyasar({ secretKey: env.MOYASAR_SECRET }),
  },
  scheduler: {
    jobs: bullmq({ redis: env.REDIS_URL }),
  },
})
```

### Interface

```ts
interface SchedulerAdapter {
  schedule(task: ScheduledTask): Promise<void>
  cancel(taskId: string): Promise<void>
}

type ScheduledTask = {
  id: string       // idempotency key — prevents double-scheduling the same work
  type: string     // task type, e.g. 'orders:auto-cancel'
  data: unknown    // typed payload per task type
  runAt: Date
  retries?: number
}
```

`task.id` is the idempotency key. Scheduling the same `id` twice must be a no-op on the adapter side.

### Packages

| Package | Scheduler |
|---|---|
| `@commerce-kit/bullmq` | BullMQ + Redis |
| `@commerce-kit/inngest` | Inngest |
| `@commerce-kit/pg-boss` | pg-boss + PostgreSQL |

---

## Built-in deferred tasks

These tasks are scheduled automatically by Commerce Kit when a scheduler adapter is configured. Timing is configurable at `createCommerce()` time.

### `orders:auto-cancel`

Cancels an order if it has not been confirmed within the configured window after placement. Scheduled at checkout. Cancelled automatically if the order advances to `confirmed` before the window expires.

```ts
createCommerce({
  scheduler: {
    jobs: bullmq({ redis: env.REDIS_URL }),
  },
  tasks: {
    'orders:auto-cancel': { after: '30m' },
  },
})
```

### `payments:reconcile`

If no payment webhook arrives within the configured window after checkout, Commerce Kit polls the payment provider directly to resolve the payment state. Prevents orders from being stuck in an unresolved payment state due to missed or delayed webhooks.

```ts
tasks: {
  'payments:reconcile': { after: '10m', retries: 3 },
}
```

### `fulfillment:reconcile`

If no fulfillment status update arrives within the configured window, Commerce Kit polls the fulfillment provider directly. Activates only when a fulfillment adapter is configured.

```ts
tasks: {
  'fulfillment:reconcile': { after: '1h', retries: 3 },
}
```

All built-in deferred tasks are dormant when no scheduler adapter is configured. Their timing defaults are intentionally conservative and should be tuned per deployment.

---

## `commerce.tasks` — recurring task API

`commerce.tasks` is the unified namespace for all named schedulable work. It is always present regardless of whether a scheduler adapter is configured.

```ts
// Run a task — call this from your cron infrastructure
await commerce.tasks.run('orders:expire-abandoned')
await commerce.tasks.run('orders:cancel-unpaid')
await commerce.tasks.run('fulfillment:sync-statuses')

// With options
await commerce.tasks.run('orders:expire-abandoned', { olderThan: '2h' })

// List all registered tasks (core + installed plugins)
await commerce.tasks.list()
```

`commerce.tasks.run` accepts only task keys registered by core or installed plugins. Calling an unregistered key is a compile-time error.

### Wiring to a cron scheduler

`commerce.tasks.run` is a plain async function. Wire it to whichever cron infrastructure the application already uses.

```ts
// node-cron
import cron from 'node-cron'
cron.schedule('0 * * * *', () => commerce.tasks.run('orders:expire-abandoned'))
cron.schedule('0 0 * * *', () => commerce.tasks.run('coupons:expire'))

// Inngest cron trigger
inngest.createFunction(
  { id: 'expire-abandoned-orders', triggers: { cron: '0 * * * *' } },
  async () => commerce.tasks.run('orders:expire-abandoned'),
)

// BullMQ repeatable job
queue.add('commerce', { task: 'coupons:expire' }, { repeat: { cron: '0 0 * * *' } })
```

Commerce Kit does not dictate the cron infrastructure. Any scheduler that can call an async function works.

### Built-in recurring tasks

| Task key | Description | Suggested schedule |
|---|---|---|
| `orders:expire-abandoned` | Transition long-stale draft orders to `cancelled` | Every hour |
| `orders:cancel-unpaid` | Cancel placed orders with no payment activity past a threshold | Every hour |
| `fulfillment:sync-statuses` | Poll fulfillment providers for pending tracking updates | Every hour |

Plugin-contributed tasks extend this table when the plugin is installed.

---

## Plugin task contributions

Plugins register tasks through a `tasks` declaration. Registered tasks become valid keys on `commerce.tasks.run`.

```ts
const couponsPlugin = plugin('coupons', {
  tasks: {
    'coupons:expire': async ({ commerce }) => {
      // expire all coupons whose endDate is in the past
    },
  },
})

// available after plugin is installed:
await commerce.tasks.run('coupons:expire')
```

Plugins may also schedule deferred tasks via the keyed `on` handler when a scheduler adapter is active:

```ts
on: {
  'orders:checkout:after': async ({ result, scheduler }) => {
    if (scheduler) {
      await scheduler.schedule({
        id: `orders:auto-cancel:${result.order.id}`,
        type: 'orders:auto-cancel',
        data: { orderId: result.order.id },
        runAt: new Date(Date.now() + 30 * 60 * 1000),
      })
    }
  },
},
```

`scheduler` is `undefined` when no scheduler adapter is configured. Plugins must guard against this before scheduling deferred work.

---

## Cross-links

- Immediate post-commit side effects via `runInBackground`: [42-hooks.md](./42-hooks.md)
- Adapter dormant activation pattern: [50-adapter-system.md](./50-adapter-system.md)
- Plugin contract for task declarations: [40-plugin-system.md](./40-plugin-system.md)
- Scheduler adapter packages: [70-packages-and-monorepo.md](./70-packages-and-monorepo.md)

## Future RFCs

- Task execution history and observability surface
- Dead-letter queue contract for failed deferred tasks
- `commerce.tasks.run` dry-run mode for CI validation
