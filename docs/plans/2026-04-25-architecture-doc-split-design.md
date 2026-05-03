# Commerce Kit Architecture Docs Split Design

**Date:** 2026-04-25
**Status:** Approved

## Goal

Replace the monolithic `prd.md` with a modular architecture document set that is easier to review, edit, and evolve, while keeping `bisuness-overview.md` as the separate business/narrative overview.

## Decisions

### 1. Canonical structure

The new canonical architecture docs live under:

```text
docs/
  architecture/
    index.md
    core-engine.md
    data-model.md
    plugin-system.md
    adapter-system.md
    type-inference.md
    packages-and-monorepo.md
    framework-adapters.md
    cli-and-tooling.md
```

### 2. Split strategy

The split is domain-based, not package-based and not a section-for-section copy of `prd.md`.

Each file owns one architectural concern:

- `index.md` — canonical entry point, architecture map, principles, links
- `core-engine.md` — scope, principles, modes, lifecycle, money, core boundaries
- `data-model.md` — entities, invariants, order state machine, persistence rules
- `plugin-system.md` — plugin contract, runtime requirements, hooks, states, extension rules
- `adapter-system.md` — database, payment, shipping, delivery, payout adapter contracts and dormant activation
- `type-inference.md` — `createCommerce()` driven type flow and compile-time guarantees
- `packages-and-monorepo.md` — monorepo layout, package map, build/versioning conventions
- `framework-adapters.md` — HTTP adapters, route handlers, SDK, webhook handling
- `cli-and-tooling.md` — config discovery, CLI commands, generation/migration flow

### 3. Source-of-truth rules

- `docs/architecture/index.md` replaces `prd.md` as the architecture entry point
- `bisuness-overview.md` remains unchanged as the narrative/business document
- architecture docs are normative, not promotional
- repetition should be minimized; files should link to each other instead of restating large sections
- current product decisions should be preserved unless explicitly revised later

### 4. Required invariants to preserve

- library, not platform
- guest in your stack
- plugin-first feature model
- single-vendor to multi-vendor without schema migration
- integer money only
- compile-time and runtime safety for invalid operations

### 5. Rewrite sequence

1. Create `docs/architecture/`
2. Write the new split docs using `prd.md` as source material
3. Add `docs/architecture/index.md` as the new canonical entry point
4. Review the split set for duplication and missing decisions
5. Remove `prd.md`
6. Keep `bisuness-overview.md` unchanged

## Doc format rules

Each architecture document should aim to include:

1. Purpose
2. Non-goals
3. Core decisions
4. Interfaces / contracts
5. Open questions (only when unresolved)
6. Future RFCs

## Non-goals for this pass

- no code scaffolding
- no package creation
- no RFC folder creation
- no architecture changes beyond clarifying the current approved direction

## Notes

This repository is currently docs-only and does not appear to be an initialized git repository in the current workspace context, so the design doc is saved locally but not committed.
