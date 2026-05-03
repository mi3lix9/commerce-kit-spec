# README and Product Overview Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a root `README.md`, move the long-form product narrative into `docs/product-overview.md`, update affected links, and remove `bisuness-overview.md`.

**Architecture:** The repository docs are split by role: `README.md` becomes the concise repo entrypoint, `docs/product-overview.md` becomes the narrative product document, and `docs/architecture/*` remains the normative technical reference. The work is a docs restructuring and rewrite, not a product-direction change.

**Tech Stack:** Markdown, repository docs structure, manual review

---

### Task 1: Create the root README scaffold

**Files:**
- Create: `README.md`

**Step 1: Create the README file**

Add the approved section structure:

```md
# Commerce Kit

## What is Commerce Kit?
## Why it exists
## Status
## Documentation
## Planned v1 scope
## Principles
## What’s not included
## Next
```

**Step 2: Fill the README with concise repo-entrypoint content**

Use the current `bisuness-overview.md` and architecture index as source material, but keep the README short and navigation-oriented.

**Step 3: Review the README**

Read `README.md` and confirm it works as a landing page rather than a long narrative.

**Step 4: Commit**

```bash
git add README.md
git commit -m "docs: add repository readme"
```

### Task 2: Create the product overview doc

**Files:**
- Create: `docs/product-overview.md`
- Source: `bisuness-overview.md`

**Step 1: Create `docs/product-overview.md`**

Rewrite the current overview into a durable docs entry under `docs/`.

**Step 2: Preserve the business narrative**

Retain the important product framing:

- library-not-platform positioning
- target users
- current market problem framing
- mode-based vendor model story
- plugin ecosystem value
- scope and deliberate exclusions

**Step 3: Avoid architecture duplication**

Do not restate technical architecture that now belongs in `docs/architecture/*`.

**Step 4: Review**

Read `docs/product-overview.md` and confirm it is clearly distinct from both `README.md` and the architecture docs.

**Step 5: Commit**

```bash
git add docs/product-overview.md
git commit -m "docs: add product overview"
```

### Task 3: Update references to the old overview location

**Files:**
- Modify: `docs/architecture/00-index.md`
- Search: repository docs for `bisuness-overview.md`

**Step 1: Find all references**

Run: `rg "bisuness-overview\.md|product-overview\.md|README\.md" .`
Expected: exact list of docs references to update

**Step 2: Update architecture and docs links**

Point old `/bisuness-overview.md` references to `/docs/product-overview.md` where appropriate.

**Step 3: Review link correctness**

Check that relative links from architecture docs resolve correctly.

**Step 4: Commit**

```bash
git add docs/architecture/00-index.md docs/product-overview.md README.md
git commit -m "docs: update overview references"
```

### Task 4: Remove the old root overview file

**Files:**
- Delete: `bisuness-overview.md`

**Step 1: Final comparison**

Confirm the needed narrative from `bisuness-overview.md` now exists in `docs/product-overview.md`.

**Step 2: Delete the old file**

Run: `rm bisuness-overview.md`
Expected: file removed successfully

**Step 3: Verify no references remain**

Run: `rg "bisuness-overview\.md" .`
Expected: no remaining matches

**Step 4: Commit**

```bash
git add -A
git commit -m "docs: replace business overview with readme and product overview"
```

### Task 5: Final docs verification

**Files:**
- Verify: `README.md`
- Verify: `docs/product-overview.md`
- Verify: `docs/architecture/00-index.md`

**Step 1: Verify file presence**

Run: `ls README.md docs/product-overview.md docs/architecture/00-index.md`
Expected: all files exist

**Step 2: Verify README role**

Ensure the README is concise and repo-facing.

**Step 3: Verify doc roles**

Ensure:
- `README.md` is the repo entrypoint
- `docs/product-overview.md` is the narrative doc
- `docs/architecture/00-index.md` is the technical entrypoint

**Step 4: Commit**

```bash
git add README.md docs/product-overview.md docs/architecture/00-index.md
git commit -m "docs: finalize repository documentation entrypoints"
```
