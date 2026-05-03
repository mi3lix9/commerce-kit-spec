# Architecture Docs Split Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace `prd.md` with a modular architecture doc set under `docs/architecture/` while preserving current product decisions and keeping `bisuness-overview.md` unchanged.

**Architecture:** The work is a documentation refactor, not a product redesign. The existing PRD is decomposed into domain-owned architecture files with `docs/architecture/index.md` becoming the canonical entry point, then `prd.md` is removed after the split is reviewed.

**Tech Stack:** Markdown, repository docs structure, manual review

---

### Task 1: Create architecture docs scaffold

**Files:**
- Create: `docs/architecture/index.md`
- Create: `docs/architecture/core-engine.md`
- Create: `docs/architecture/data-model.md`
- Create: `docs/architecture/plugin-system.md`
- Create: `docs/architecture/adapter-system.md`
- Create: `docs/architecture/type-inference.md`
- Create: `docs/architecture/packages-and-monorepo.md`
- Create: `docs/architecture/framework-adapters.md`
- Create: `docs/architecture/cli-and-tooling.md`

**Step 1: Create the directory and empty files**

Run: `mkdir -p docs/architecture`
Expected: directory exists with no error

**Step 2: Add headings and purpose sections**

Write minimal initial headings to each file so ownership is explicit before moving content.

**Step 3: Review file list**

Run: `ls docs/architecture`
Expected: the 8 architecture docs exist

**Step 4: Commit**

If the repo is initialized:

```bash
git add docs/architecture
git commit -m "docs: scaffold architecture documentation"
```

If git is unavailable, skip and continue.

### Task 2: Write the canonical architecture index

**Files:**
- Modify: `docs/architecture/index.md`
- Source: `bisuness-overview.md`
- Source: `prd.md`

**Step 1: Write the short overview**

Include:
- what Commerce Kit is
- draft status
- design principles
- architecture map linking to every architecture file

**Step 2: Keep it concise**

Do not duplicate full PRD sections. Limit this file to navigation and foundational framing.

**Step 3: Review**

Read `docs/architecture/index.md` and confirm every downstream file is linked.

**Step 4: Commit**

```bash
git add docs/architecture/index.md
git commit -m "docs: add architecture index"
```

### Task 3: Move core platform decisions into `core-engine.md`

**Files:**
- Modify: `docs/architecture/core-engine.md`
- Source: `prd.md:9-112`
- Source: `bisuness-overview.md:7-138`

**Step 1: Add scope and principles**

Capture:
- library not platform
- guest in your stack
- plugin-first rule
- mode-based vendor model
- headless-only scope
- permanent out-of-scope items

**Step 2: Add engine responsibilities**

Summarize products, orders, payments, pricing/tax, shipping/delivery, marketplace support.

**Step 3: Add non-goals and v1 boundaries**

Keep out anything that belongs in data model, adapters, CLI, or framework transport docs.

**Step 4: Review**

Confirm this file reads as product architecture, not package inventory.

**Step 5: Commit**

```bash
git add docs/architecture/core-engine.md
git commit -m "docs: define core engine architecture"
```

### Task 4: Move entities and state machine into `data-model.md`

**Files:**
- Modify: `docs/architecture/data-model.md`
- Source: `prd.md:113-165`

**Step 1: Document conventions**

Include ID generation, deletion policy, money field rules, and adapter-owned column naming.

**Step 2: Document entities**

Move `vendor`, `product`, `productVariant`, `order`, `vendorOrder`, `orderItem`, `payment`, and `orderTransition`.

**Step 3: Document invariants**

Call out immutable snapshots, append-only ledger behavior, and non-deletable financial records.

**Step 4: Document the base state machine**

Include the base order statuses and note plugin extension points.

**Step 5: Commit**

```bash
git add docs/architecture/data-model.md
git commit -m "docs: add data model architecture"
```

### Task 5: Move plugin contract into `plugin-system.md`

**Files:**
- Modify: `docs/architecture/plugin-system.md`
- Source: `prd.md:166-234`

**Step 1: Add the plugin interface**

Document `CommercePlugin` and every extension surface.

**Step 2: Add the two-layer dependency model**

Document `peerDependencies` vs runtime `requires[]`.

**Step 3: Add execution and safety rules**

Cover init order, hook order, forbidden behaviors, and plugin state extension rules.

**Step 4: Commit**

```bash
git add docs/architecture/plugin-system.md
git commit -m "docs: define plugin system architecture"
```

### Task 6: Move adapter contracts into `adapter-system.md`

**Files:**
- Modify: `docs/architecture/adapter-system.md`
- Source: `prd.md:235-449`

**Step 1: Add packaging rules**

Explain which adapter interfaces are bundled in core and which provider implementations are separate packages.

**Step 2: Add adapter interfaces**

Document database, payment, shipping, delivery, storage, and payout contracts.

**Step 3: Add dormant activation model**

Explain how shipping/delivery/payout surface area appears only when configured.

**Step 4: Add comparison guidance**

Keep the shipping vs delivery distinction because it is architectural, not just operational.

**Step 5: Commit**

```bash
git add docs/architecture/adapter-system.md
git commit -m "docs: define adapter system architecture"
```

### Task 7: Move type guarantees into `type-inference.md`

**Files:**
- Modify: `docs/architecture/type-inference.md`
- Source: `prd.md:450-492`

**Step 1: Document the type flow**

Explain config → server instance → route handlers → client inference.

**Step 2: Document compile-time guarantees**

Include missing plugin APIs, mode-gated vendor APIs, dormant adapter APIs, invalid states, and branded IDs.

**Step 3: Capture TypeScript requirements**

Include TS 5.0+ and TS-only constraints.

**Step 4: Commit**

```bash
git add docs/architecture/type-inference.md
git commit -m "docs: define type inference architecture"
```

### Task 8: Move monorepo and package mapping into `packages-and-monorepo.md`

**Files:**
- Modify: `docs/architecture/packages-and-monorepo.md`
- Source: `prd.md:493-590`
- Source: `prd.md:740-778`

**Step 1: Add tooling and build conventions**

Document Turborepo, Bun, tsdown, Vitest, Changesets, ESM-only output, and runtime targets.

**Step 2: Add directory structure**

Document the proposed monorepo tree.

**Step 3: Add package map**

Separate v1, post-v1, and community package categories.

**Step 4: Add versioning rules**

Document semver, pre-v1 release tiers, and release workflow.

**Step 5: Commit**

```bash
git add docs/architecture/packages-and-monorepo.md
git commit -m "docs: add monorepo architecture"
```

### Task 9: Move HTTP exposure into `framework-adapters.md`

**Files:**
- Modify: `docs/architecture/framework-adapters.md`
- Source: `prd.md:779-1058`

**Step 1: Document architecture layers**

Explain core logic vs framework adapters vs client SDK.

**Step 2: Document route handlers and validation**

Include canonical route ownership, validation behavior, and `resolveContext`.

**Step 3: Document framework mappings**

Keep the usage examples for Hono, Elysia, Express, Fastify, and Next.js.

**Step 4: Document webhooks and errors**

Include raw body handling, verification flow, idempotency, and HTTP status mapping.

**Step 5: Commit**

```bash
git add docs/architecture/framework-adapters.md
git commit -m "docs: define framework adapter architecture"
```

### Task 10: Move CLI and config conventions into `cli-and-tooling.md`

**Files:**
- Modify: `docs/architecture/cli-and-tooling.md`
- Source: `prd.md:652-739`

**Step 1: Document config file conventions**

Include default paths, overrides, and required export shape.

**Step 2: Document CLI commands**

Include `init`, `generate`, `migrate`, `add`, and `check`.

**Step 3: Separate v1 vs post-v1 commands**

Keep future commands clearly marked.

**Step 4: Commit**

```bash
git add docs/architecture/cli-and-tooling.md
git commit -m "docs: define CLI and tooling architecture"
```

### Task 11: Review cross-links and duplication

**Files:**
- Modify: `docs/architecture/*.md`

**Step 1: Read all architecture docs**

Check for repeated package lists, repeated principles, and misplaced details.

**Step 2: Trim duplication**

Keep one canonical home per concern and replace repeated text with short references.

**Step 3: Verify navigation**

Ensure every architecture file is linked from `index.md` and references point to existing files.

**Step 4: Commit**

```bash
git add docs/architecture
git commit -m "docs: tighten architecture cross references"
```

### Task 12: Replace the old PRD

**Files:**
- Delete: `prd.md`

**Step 1: Final comparison**

Confirm no critical content from `prd.md` was lost or left unmapped.

**Step 2: Remove the old file**

Run: `rm prd.md`
Expected: file removed successfully

**Step 3: Verify final docs set**

Run: `ls docs/architecture`
Expected: canonical doc set remains and `prd.md` no longer exists

**Step 4: Commit**

```bash
git add -A
git commit -m "docs: replace PRD with modular architecture docs"
```
