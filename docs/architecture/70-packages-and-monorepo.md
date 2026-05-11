# Packages and Monorepo

## Purpose

Define the canonical package layout, tooling stack, build conventions, package map, and release/versioning rules for Commerce Kit.

## Non-goals

- HTTP route behavior and framework-specific adapter usage
- Config discovery, config file shape, and CLI command behavior
- Core runtime behavior outside package/build/release boundaries

## Core decisions

### Tooling stack

| Tool | Role |
|---|---|
| Turborepo | Task orchestration and caching |
| Bun | Package manager, scripts, and tests |
| tsdown | Library builds |
| Vitest | Test runner |
| Changesets | Independent package versioning |

The supported first-party development toolchain uses Bun rather than pnpm.

### Build conventions

- ESM only
- No CommonJS output
- `type: "module"` in every package
- Node.js 18+ runtime baseline for published packages
- Bun 1.0+ required for the first-party monorepo workflow, scripts, and tests
- Single public entry point per package: `src/index.ts`
- Zero runtime dependencies in `commerce-kit` core
- No circular dependencies

### Runtime and tooling requirements

- Published runtime packages target Node.js 18+.
- Framework adapters and provider packages may also run in Bun-compatible server environments when their underlying framework/provider supports it.
- The first-party repository uses Bun for package management, task scripts, local testing, and CLI execution examples.
- `@commerce-kit/cli` is documented with `bunx` as the canonical runner, but that is a tooling contract rather than a statement that application runtime requires Bun.

Canonical library build shape:

```ts
export default defineConfig({
  entry: ["src/index.ts"],
  format: "esm",
  dts: true,
  clean: true,
  platform: "neutral",
})
```

### Monorepo layout

```text
commerce-kit/
├── packages/
│   ├── core/                    ← "commerce-kit"
│   ├── moyasar/                 ← "@commerce-kit/moyasar"
│   ├── tabby/                   ← "@commerce-kit/tabby"
│   ├── better-auth/             ← "@commerce-kit/better-auth"
│   ├── pricing-rules/           ← "@commerce-kit/pricing-rules"
│   ├── pricing-rules-dynamic/   ← "@commerce-kit/pricing-rules-dynamic"
│   ├── coupons/                 ← "@commerce-kit/coupons"
│   ├── aramex/                  ← "@commerce-kit/aramex"
│   ├── smsa/                    ← "@commerce-kit/smsa"
│   ├── marsool/                 ← "@commerce-kit/marsool"
│   ├── jahez/                   ← "@commerce-kit/jahez"
│   ├── marketplace/             ← "@commerce-kit/marketplace"
│   ├── vendor-payouts/          ← "@commerce-kit/vendor-payouts"
│   ├── hono/                    ← "@commerce-kit/hono"
│   ├── elysia/                  ← "@commerce-kit/elysia"
│   ├── express/                 ← "@commerce-kit/express"
│   ├── fastify/                 ← "@commerce-kit/fastify"
│   ├── nextjs/                  ← "@commerce-kit/nextjs"
│   ├── client/                  ← "@commerce-kit/client"
│   └── cli/                     ← "@commerce-kit/cli"
├── internal/
│   ├── tsconfig/
│   ├── eslint-config/
│   └── test-utils/
├── examples/
├── docs/
├── turbo.json
├── bunfig.toml
└── package.json
```

### Package map

#### Core packages

| Package | Type |
|---|---|
| `commerce-kit` | core + bundled `drizzleAdapter` + bundled adapter interfaces for payment/fulfillment/storage/payout/scheduler |
| `@commerce-kit/moyasar` | payment adapter |
| `@commerce-kit/tabby` | payment adapter (BNPL) |
| `@commerce-kit/better-auth` | auth bridge |
| `@commerce-kit/pricing-rules` | plugin (static rules) |
| `@commerce-kit/pricing-rules-dynamic` | plugin (DB-backed rules) |
| `@commerce-kit/coupons` | plugin |
| `@commerce-kit/aramex` | fulfillment adapter (carrier shipping) |
| `@commerce-kit/smsa` | fulfillment adapter (carrier shipping) |
| `@commerce-kit/marsool` | fulfillment adapter (local delivery) |
| `@commerce-kit/jahez` | fulfillment adapter (local delivery) |
| `@commerce-kit/marketplace` | major domain plugin for vendor, `vendorOrder`, vendor ownership, and marketplace workflows |
| `@commerce-kit/vendor-payouts` | marketplace-adjacent plugin using the core payout adapter interface |
| `@commerce-kit/hono` | framework adapter |
| `@commerce-kit/elysia` | framework adapter |
| `@commerce-kit/express` | framework adapter |
| `@commerce-kit/fastify` | framework adapter |
| `@commerce-kit/nextjs` | framework adapter |
| `@commerce-kit/client` | fetch-based type-safe client runtime, including `createCommerceClient` |
| `@commerce-kit/cli` | CLI |

#### Planned packages

| Package | Type |
|---|---|
| `@commerce-kit/cart` | enhanced cart plugin (wishlists, saved carts) |
| `@commerce-kit/customers` | plugin |
| `@commerce-kit/inventory` | plugin |
| `@commerce-kit/localization` | plugin |
| `@commerce-kit/reviews` | plugin |
| `@commerce-kit/digital-products` | plugin |
| `@commerce-kit/subscriptions` | plugin |
| `@commerce-kit/orpc` | oRPC procedures + OpenAPI |
| `@commerce-kit/bullmq` | scheduler adapter (BullMQ + Redis) |
| `@commerce-kit/inngest` | scheduler adapter (Inngest) |
| `@commerce-kit/pg-boss` | scheduler adapter (pg-boss + PostgreSQL) |

#### Community packages

| Package | Type |
|---|---|
| `@commerce-kit/stripe` | payment adapter |
| `@commerce-kit/dhl` | fulfillment adapter (carrier shipping) |
| `@commerce-kit/toyou` | fulfillment adapter (local delivery) |
| `@commerce-kit/hyperpay` | payout adapter |
| `@commerce-kit/r2` | storage adapter |

`@commerce-kit/marketplace` is the first-class package that owns the marketplace domain boundary. Related packages such as `@commerce-kit/vendor-payouts` may build on marketplace workflows or optional payout adapters, but they do not replace marketplace as the source of truth for vendor topology.

The package inventory follows the adapter-system contract: provider-neutral adapter interfaces and the bundled `drizzleAdapter` live in `commerce-kit`, while provider implementations, the fetch client runtime (`@commerce-kit/client`), and optional domain packages ship separately.

## Versioning and release rules

### Semver contract

- Patch: bug fixes with no API changes
- Minor: backward-compatible new features
- Major: breaking changes and a written migration guide

Any change to the `CommercePlugin` interface or core database schema is a major version bump.

### Release workflow

- Every PR includes a Changesets file
- Merge to `main` opens a Changesets version PR
- Merging that PR publishes to npm
- Packages are versioned independently

### Internal build migration rule

`tsdown` uses Rolldown internally. A future migration aligned with Rolldown stability is treated as an internal implementation detail and not a semver-visible product change by itself.

## Interfaces and cross-links

- For framework HTTP exposure and SDK behavior, see `80-framework-adapters.md`
- For config conventions and CLI workflow, see `90-cli-and-tooling.md`
- For plugin and adapter contracts, see the dedicated architecture docs for those domains

## Future RFCs

- Any change to package boundaries or release policy
- Any new first-party package not already listed in the approved package map
