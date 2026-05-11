# CLI and Tooling

## Purpose

Define config path conventions, required config file shape, recommended project structure, and the v1 CLI command contract.

## Non-goals

- Monorepo package ownership, build stack, and release/versioning policy
- Framework adapter HTTP behavior and webhook transport details
- Core runtime semantics outside config and CLI workflows

## Core decisions

### Config path convention

The default Commerce Kit config location is `lib/commerce.ts`.

This follows the same singleton-placement convention as `lib/auth.ts` and `lib/db.ts`.

### CLI config discovery order

```text
lib/commerce.ts
lib/commerce/index.ts
src/lib/commerce.ts
src/commerce.ts
utils/commerce.ts
commerce.config.ts
```

### Config path overrides

- Package-level override in `package.json`:

```json
{ "commerce-kit": { "configPath": "app/server/commerce.ts" } }
```

- Per-command override via CLI flag: `--config path/to/commerce.ts`

## Config file shape

The config file must export a named `commerce` constant.

```ts
export const commerce = createCommerce({ ... })
export type Types = InferCommerceTypes<typeof commerce>
```

This is the contract the CLI uses for discovery and type-driven generation.

## Recommended project structure

```text
lib/
├── commerce.ts
├── db.ts
└── auth.ts
routes/
├── orders.ts
└── webhooks.ts
schema.ts
src/
└── index.ts
```

`src/lib/commerce.ts` remains a supported discovery path, but `lib/commerce.ts` is the canonical default.

## CLI package

`@commerce-kit/cli` is a separate published package.

- It can be installed as a dev dependency or run via `bunx`
- It uses Bun's native TypeScript execution to import the config file without a compile step

CLI/tooling requirements are distinct from application runtime requirements:

- application packages target the runtime baseline documented in `70-packages-and-monorepo.md`
- the first-party CLI workflow assumes Bun for local execution and examples
- framework/server runtime choice remains an application concern unless a specific adapter or provider package documents additional constraints

Client packaging is separate as well: application code imports the fetch client runtime from `@commerce-kit/client`, while the Commerce Kit config in `lib/commerce.ts` remains the source of server-side types used for inference.

## v1 commands

### `commerce-kit init`

Scaffolds Commerce Kit into an existing project.

- Detects the framework
- Creates `lib/commerce.ts`
- Installs selected plugins and adapters
- Runs `generate` automatically

### `commerce-kit generate`

Generates Commerce Kit-managed Drizzle schema from core, installed plugins, and any adapter-activated schema owned by active core interfaces.

- Reads the Commerce Kit config
- Collects tables and schema extensions from core, installed plugins, and activated optional adapter surfaces
- Merges into an existing schema without touching custom tables
- Marks Commerce Kit-managed tables with `// commerce-kit:managed`

### `commerce-kit migrate`

Runs generation first, then delegates migration execution to Drizzle Kit.

- Never auto-migrates on application startup
- Supports `--dry-run`
- Supports `--yes`

### `commerce-kit add <plugin>`

Performs the approved automation sequence:

1. Install the npm package
2. Add plugin or adapter imports and scaffolded config to `lib/commerce.ts`
3. Run `generate`
4. Run `migrate`
5. Warn about missing runtime dependencies; do not install them silently

Adding `@commerce-kit/marketplace` is the canonical expansion flow from a simple store into marketplace topology. The tooling story is migration-assisted, not migration-free.

Expected marketplace expansion flow:

1. Install `@commerce-kit/marketplace`
2. Scaffold marketplace plugin configuration into `lib/commerce.ts`
3. Run `generate` to add marketplace-managed tables and extensions
4. Run `migrate` to apply the schema changes
5. Run marketplace validation/backfill helpers to attach vendor ownership where required
6. Report any historical orders or products that need manual attribution before full marketplace workflows are enabled

### `commerce-kit check`

Validates the full setup for local verification and CI.

- Config discovery
- Installed plugins
- Runtime requirements
- Schema sync

When marketplace is installed, `check` should also validate the marketplace expansion boundary: required tables exist, vendor ownership backfills are complete enough for the selected workflows, and any unresolved historical attribution gaps are surfaced explicitly.

## Planned commands

- `commerce-kit seed`
- `commerce-kit studio`
- `commerce-kit types`

## Source mapping and boundaries

- Tooling stack, monorepo layout, package map, and release rules belong in `70-packages-and-monorepo.md`
- HTTP adapters, request validation, `resolveContext`, client SDK, and webhooks belong in `80-framework-adapters.md`
- This document owns only config conventions and CLI behavior

## Future RFCs

- Any change to config discovery order
- Any new code-generation output contract
- Any new CLI command beyond the approved list
