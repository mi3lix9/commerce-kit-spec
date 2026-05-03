# Docs Examples Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a short proposed-usage example to `README.md` and a fuller starter example guide under `docs/examples/`.

**Architecture:** The README gets a compact example that shows the overall `createCommerce(...)` shape. A separate `docs/examples/getting-started.md` file carries the fuller illustrative example with config, adapter slots, framework mounting, and request flow, while staying explicit that the API shape is proposed rather than shipped.

**Tech Stack:** Markdown, TypeScript example snippets, repository docs structure

---

### Task 1: Add a quick example to the README

**Files:**
- Modify: `README.md`

**Step 1: Add a `## Quick example` section**

Place it after the project introduction sections and before the deeper documentation references if that reads best.

**Step 2: Add one compact TypeScript example**

The snippet should show the intended shape of:

- `createCommerce({ ... })`
- `database`
- `payments`
- `shipping`
- `delivery`
- `plugins`
- `mode`

Use one provider per adapter category to keep the snippet readable.

**Step 3: Add one short framing note**

State that the example reflects the proposed v1 usage shape described by the docs.

**Step 4: Review**

Read `README.md` and confirm the example is short, readable, and does not overwhelm the landing page.

### Task 2: Create the full example guide

**Files:**
- Create: `docs/examples/getting-started.md`

**Step 1: Create `docs/examples/` if needed**

Run: `mkdir -p docs/examples`
Expected: directory exists with no error

**Step 2: Add the guide structure**

Include sections for:

- what this example demonstrates
- proposed project structure
- proposed `lib/commerce.ts`
- example framework mounting
- example request flow
- what each adapter/plugin slot is for
- optional/dormant adapter note

**Step 3: Write the main TypeScript examples**

The guide should show a realistic starter app shape that includes all major adapter categories at least once:

- database adapter
- payment adapter
- shipping adapter
- delivery adapter
- plugin
- mode

Keep it illustrative rather than pretending it is already runnable production code.

**Step 4: Review**

Read `docs/examples/getting-started.md` and confirm it teaches the shape clearly without duplicating architecture docs.

### Task 3: Link the example guide from README

**Files:**
- Modify: `README.md`

**Step 1: Add the example guide to the Documentation section**

Include a link to `./docs/examples/getting-started.md`.

**Step 2: Check navigation flow**

Ensure README now gives a clear progression:

- product overview
- getting started example
- architecture index

**Step 3: Review**

Make sure the README still feels like a concise repo entrypoint.

### Task 4: Final docs verification

**Files:**
- Verify: `README.md`
- Verify: `docs/examples/getting-started.md`

**Step 1: Verify file presence**

Run: `ls README.md docs/examples/getting-started.md`
Expected: both files exist

**Step 2: Verify links**

Run: `rg "getting-started\.md|Quick example|createCommerce\(" README.md docs/examples/getting-started.md`
Expected: README includes the example and links to the full guide; the full guide includes the intended example shape

**Step 3: Verify role separation**

Confirm:

- README has a compact example
- the guide has the fuller example
- architecture docs remain technical reference, not usage tutorial
