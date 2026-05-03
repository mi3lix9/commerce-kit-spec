# Single-Store-First Marketplace Plugin Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rewrite the Commerce Kit docs so simple-store is the core default and marketplace is a first-class plugin-based expansion with an easy-migration story.

**Architecture:** This is a boundary rewrite, not a cosmetic wording pass. Product-facing docs, architecture docs, and examples all need to align around a new core/plugin split: core single-store commerce primitives by default, marketplace concepts only when the marketplace plugin is installed.

**Tech Stack:** Markdown, repository docs structure, architectural documentation

---

### Task 1: Rewrite the product-facing promise

**Files:**
- Modify: `README.md`
- Modify: `docs/product-overview.md`

**Step 1: Update README positioning**

Remove or replace language that implies:

- same schema from day one
- no migration
- single-vendor/multi-vendor mode flip as the headline story

Replace it with:

- simple-store-first
- marketplace as plugin expansion
- easy migration when expansion is needed

**Step 2: Update product overview narrative**

Reframe growth/migration language so it matches the new stance: marketplace is first-class but opt-in; migration is guided and expected, not denied.

**Step 3: Review**

Confirm both docs now tell the same product story.

### Task 2: Rewrite the core architecture boundary

**Files:**
- Modify: `docs/architecture/00-index.md`
- Modify: `docs/architecture/10-core-engine.md`
- Modify: `docs/architecture/20-data-model.md`
- Modify: `docs/architecture/40-plugin-system.md`

**Step 1: Update architecture index**

Remove shared-schema/no-migration/vendor-in-core assumptions from the top-level architecture framing.

**Step 2: Update core engine doc**

Remove `mode` as a base-core assumption and define marketplace as plugin-based expansion.

**Step 3: Update core data model doc**

Remove `vendor`, `vendorOrder`, and base `product.vendorId` from the core model. Keep the doc limited to the simple-store core entities.

**Step 4: Update plugin system doc**

Clarify that plugins can own major domain expansions such as marketplace, not only small feature add-ons.

**Step 5: Review**

Confirm the core docs no longer assume vendor topology exists by default.

### Task 3: Add the marketplace plugin architecture doc

**Files:**
- Create: `docs/architecture/45-marketplace-plugin.md`

**Step 1: Create the new doc**

Document the marketplace plugin as the owner of:

- `vendor`
- `vendorOrder`
- vendor ownership on products
- vendor APIs/routes
- order splitting by vendor
- payouts/commissions/onboarding boundaries

**Step 2: Document migration stance**

Describe the promise as easy migration with tooling rather than no migration.

**Step 3: Link it into the architecture map**

Update `docs/architecture/00-index.md` to include the new file in the correct ordered position.

### Task 4: Rewrite downstream docs to match plugin-gated marketplace

**Files:**
- Modify: `docs/architecture/60-type-inference.md`
- Modify: `docs/architecture/70-packages-and-monorepo.md`
- Modify: `docs/architecture/80-framework-adapters.md`
- Modify: `docs/architecture/90-cli-and-tooling.md`

**Step 1: Update type inference doc**

Replace mode-gated marketplace/vendor language with plugin-gated marketplace surface language.

**Step 2: Update package map**

Introduce `@commerce-kit/marketplace` as the first-class package owning the marketplace domain. Reposition related marketplace packages under or alongside it as needed.

**Step 3: Update framework adapters doc**

Move vendor routes out of the base route set and describe them as marketplace-installed routes.

**Step 4: Update CLI/tooling doc**

Describe the marketplace expansion/migration flow as part of the tooling story.

### Task 5: Rewrite examples to match the new boundary

**Files:**
- Modify: `README.md`
- Modify: `docs/examples/getting-started.md`

**Step 1: Update README example/story**

Ensure the core example stays simple-store-first and does not imply vendor topology by default.

**Step 2: Update getting-started guide**

Keep the starter example core-only by default, while explaining how marketplace would be added later as a plugin.

**Step 3: Review**

Confirm examples no longer contradict the rewritten architecture.

### Task 6: Final verification

**Files:**
- Verify: `README.md`
- Verify: `docs/product-overview.md`
- Verify: `docs/architecture/*.md`
- Verify: `docs/examples/getting-started.md`

**Step 1: Verify architecture map**

Check that `45-marketplace-plugin.md` is linked from `00-index.md` and that numbering/order remain coherent.

**Step 2: Verify old promise removal**

Search for phrases like `same schema`, `no migration`, and mode-based vendor growth claims that should no longer be canonical.

**Step 3: Verify consistency**

Confirm all docs now say:

- core is simple-store-first
- marketplace is plugin-based
- migration is easy, not absent
