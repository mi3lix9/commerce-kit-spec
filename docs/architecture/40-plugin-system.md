# Plugin System

## Purpose

This document defines the Commerce Kit plugin contract. It is the canonical source for `CommercePlugin`, runtime requirements, hook behavior, plugin-owned state extensions, and forbidden plugin behavior.

Adapter activation, packaging boundaries, and `createCommerce()` type inference are defined in [50-adapter-system.md](./50-adapter-system.md) and [60-type-inference.md](./60-type-inference.md).

## Non-goals

- Defining core entities or persistence invariants
- Defining adapter interfaces
- Repeating framework transport or webhook routing details

## Core decisions

- Plugins are the primary extension model for Commerce Kit features.
- Plugins may own bounded features or major domain expansions.
- Plugins may extend schema, lifecycle hooks, APIs, order states, client APIs, and verified webhook handling.
- Plugins must declare runtime dependencies explicitly via `requires[]`.
- Plugin behavior is normative and ordered: declaration order controls initialization and hook execution.
- Plugins may extend the order state machine only through `states`.

## `CommercePlugin` contract

```ts
interface CommercePlugin {
  id: string
  name: string
  version: string

  requires?: RuntimeRequirement[]
  schema?: PluginSchema
  hooks?: PluginHooks
  api?: PluginApi
  states?: PluginStates
  client?: PluginClientExtension

  webhooks?: Record<
    string,
    (ctx: RequestContext, event: WebhookEvent) => Promise<void>
  >

  onInit?: (ctx: CommerceContext) => Promise<void>
  onTeardown?: (ctx: CommerceContext) => Promise<void>
}
```

### Required fields

- `id` must be unique and kebab-case.
- `version` must be semver and must match the published npm package version.

### Extension surfaces

1. **`schema`** — adds tables and extends existing entities.
2. **`hooks`** — lifecycle middleware around core operations.
3. **`api`** — adds typed namespaces to `commerce.*`.
4. **`states`** — adds order states and declared transitions.
5. **`client`** — extends the generated client surface.
6. **`webhooks`** — handles verified webhook events only.
7. **`onInit` / `onTeardown`** — startup and teardown lifecycle.

`api` is a server-side extension surface first. Any HTTP exposure of plugin APIs is mediated by the core route layer and framework adapters rather than raw plugin-owned router registration.

If a plugin needs HTTP exposure, it must do so through a declarative route contribution surface owned by Commerce Kit rather than by mounting its own framework router. Normatively, a plugin may contribute route definitions consisting of:

- an API namespace it already owns under `api`
- unique HTTP method/path pairs
- a reference to the plugin-owned typed handler or operation that core should invoke

Framework adapters only install route definitions that pass this core-owned contract.

Plugin-owned route handlers and plugin API handlers execute through the same Commerce Kit operation pipeline as core handlers. When a core operation opens a database transaction, participating plugin hooks and plugin-owned handlers for that operation run against the same transaction boundary rather than spawning independent commits.

Collision policy is normative:

- each plugin API namespace must be unique
- each plugin client namespace or client-surface key must be unique after composition with core and other installed plugins
- each plugin-contributed route signature (`method + path`) must be unique after composition with core and other installed plugins
- collisions must fail during initialization; plugins must not rely on override order or shadowing

Plugins are not limited to minor add-ons. A plugin may own a major domain boundary when that capability is optional to the core product model. Marketplace is the canonical example: it can introduce its own entities, APIs, workflows, and state extensions without making vendor topology a base-core assumption.

## Dependency model

Commerce Kit uses two dependency layers.

### Package installation

`peerDependencies` ensure the required packages are installed and version-resolved by the package manager.

### Runtime requirements

`requires[]` is validated during `createCommerce()`.

```ts
type RuntimeRequirement =
  | { type: "plugin"; pluginId: string; version?: string }
  | { type: "config"; key: string; value?: unknown; optional?: boolean }
  | { type: "adapter"; adapter: "payment" | "storage" | "fulfillment" | "payout" }
```

Use `requires[]` for runtime topology and config guarantees, not for package installation.

## Hooks

### Hook model

- Hooks are middleware-style.
- Before hooks may abort execution by throwing.
- After hooks run after the core handler has produced its result, but before the enclosing Commerce Kit operation is finalized.
- If the triggering operation is transactional, after hooks run inside that same transaction and may still cause the operation to fail and roll back.
- An after hook for a non-transactional operation runs after the core handler result and may fail the overall operation response because there is no commit boundary to preserve separately.
- A before hook must either call `next()` or throw.

### Execution rules

- Plugins initialize in declaration order.
- Required plugins must appear earlier in the plugin array.
- Hook execution follows declaration order.
- Teardown runs through the plugin lifecycle contract; plugin authors must not assume unordered shutdown.
- Before and after hooks participating in a core write operation share that operation's transaction boundary.

Normative transaction boundary rule:

- `after` means after the core handler logic, not after database commit.
- Commerce Kit must not commit a transactional operation until all participating after hooks complete successfully.
- Plugin authors must treat after hooks as part of the same atomic operation when a transaction exists.
- If a plugin needs post-commit or best-effort side effects, that work must be modeled through a separate async job/outbox pattern rather than a transactional after hook.

## Plugin-owned state extensions

Plugins may extend the order state machine with additional states and transitions. This is the only valid extension point for state-machine mutation.

Rules:

- Base states remain defined by the core data model.
- Plugin states must be declared in `states`.
- Transitions must be declared, not implied.
- Invalid transitions must always fail at runtime. Compile-time narrowing is expected only when transition inputs are statically visible to TypeScript.

The base state machine lives in the data model documentation; this file only defines the plugin extension contract.

## Plugin-owned data access

Plugins own their data through declared schema surfaces and the Commerce Kit operation context.

- Plugin-owned tables and entity extensions must be declared through `schema`.
- Plugin reads and writes happen through the Commerce Kit context/database adapter contract, not through raw out-of-band database clients.
- Plugin hooks and plugin API handlers may read or write plugin-owned tables, and may update declared plugin-owned extensions on core entities, as part of the same operation context.
- When invoked inside a core transactional operation, plugin reads/writes and both before/after hooks participate in that same transaction and succeed or fail with the enclosing core operation.
- Plugins must not assume a separate private transaction boundary unless they explicitly open a nested adapter transaction where the adapter supports it and the surrounding operation contract allows it.

## Major domain expansion rule

When a plugin owns a major domain expansion, it is responsible for documenting:

- the plugin-owned entities and relations
- any extension of core entities
- plugin-owned APIs and any route contributions exposed through framework adapters
- operational boundaries and lifecycle rules
- migration and tooling expectations for adopting the expansion

Core documents should name the boundary but should not inline the expansion's schema or workflows as if they are always present.

If a major expansion depends on an optional adapter interface such as payout, the plugin owns the business workflow while the adapter system owns the provider contract.

## Webhook handling rule

Plugins never receive raw inbound webhook payloads. Adapters verify signatures and parse provider payloads first. Plugin webhook handlers run only after Commerce Kit has produced a verified `WebhookEvent`.

Plugin webhook handlers are event consumers only:

- they cannot replace adapter signature verification
- they cannot intercept unverified transport payloads
- they operate on the normalized verified event shape selected by the active adapter contract

### Webhook registration and dispatch

- `webhooks` keys are normalized verified event type strings produced by the active adapter contract after `verifyWebhook(...)` succeeds.
- A plugin registers handlers only for the event types it wants to consume; missing keys mean no subscription.
- After adapter verification, parsing, and core idempotency/event handling, Commerce Kit dispatches the verified event to matching plugin handlers in plugin declaration order.
- If multiple plugins subscribe to the same verified event type, each matching handler runs; registration is additive, not exclusive.
- A plugin must not assume provider-native event names if the adapter contract normalizes them into Commerce Kit event types.
- Dispatch is sequential in plugin declaration order.
- If one plugin webhook handler fails, Commerce Kit records that failure, stops dispatch to later plugin handlers for that event, and treats the webhook as unsuccessfully processed.
- A webhook request returns HTTP 200 only after adapter verification, core event handling, and plugin dispatch complete successfully.
- If plugin dispatch fails after verification, Commerce Kit returns a non-2xx error so the upstream provider or operator-visible retry flow can retry according to adapter/provider delivery behavior.
- Retry ownership for webhook delivery remains with the upstream provider and the adapter-facing webhook endpoint contract; plugins do not schedule transport-level retries themselves.

## Forbidden behavior

Plugins must not:

- import another plugin's internal functions directly
- issue raw database queries
- register a `before` hook that neither calls `next()` nor throws
- register a conflicting plugin `id`
- mutate the state machine outside the `states` declaration

## Future RFCs

- Additional plugin capability surfaces beyond the current contract
- Changes to `CommercePlugin` itself, which are major-version changes
