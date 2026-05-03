# Client API Documentation Plan

> Superseded note: the canonical client API direction has moved into `docs/architecture/60-type-inference.md`, `docs/architecture/80-framework-adapters.md`, and `docs/examples/getting-started.md`. Those docs now use the domain-first, object-only API with `orders.checkout(...)`, unified fulfillment, and high-level order payment workflows.

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Document the approved Commerce Kit client API before any package or runtime implementation exists.

**Architecture:** Keep this phase documentation-only. Update the architecture and example docs so they describe the unified `commerce.domain.method(...)` API, capability-based method omission, and the Drizzle-inspired query model without creating `packages/`, `package.json`, or runtime source files.

**Tech Stack:** Markdown docs, architecture RFCs, examples

---

### Task 1: Document the unified client surface in type inference docs

**Files:**
- Modify: `docs/architecture/60-type-inference.md`
- Reference: `docs/plans/2026-04-26-client-api-design.md`

**Step 1: Write the missing doc assertions**

Create a short checklist for this file:

- shared server/client API shape
- capability-based method omission
- plugin- and adapter-gated namespaces
- domain-specific inferred query types

**Step 2: Review the current doc to verify the gap**

Read: `docs/architecture/60-type-inference.md`
Expected: the document mentions `createCommerceClient<typeof commerce>()` but does not yet define the domain-first method surface in enough detail.

**Step 3: Update the doc minimally**

Add a focused section that describes:

- `commerce.domain.method(...)` as the canonical mental model
- omitted methods for immutable domains
- inferred `where` and `orderBy` shapes as part of namespace inference

**Step 4: Review the edited markdown**

Expected: the new section reads as an architecture contract, not an implementation tutorial.

**Step 5: Commit**

```bash
git add docs/architecture/60-type-inference.md
git commit -m "docs: define inferred client api surface"
```

### Task 2: Document transport mapping and runtime parity in framework adapter docs

**Files:**
- Modify: `docs/architecture/80-framework-adapters.md`
- Reference: `docs/plans/2026-04-26-client-api-design.md`

**Step 1: Write the missing doc assertions**

Create a short checklist for this file:

- same public shape across server and HTTP runtimes
- route mapping for standard CRUD-like methods
- action route mapping for workflow methods like `orders.transition`
- `list` and `search` query expectations

**Step 2: Review the current doc to verify the gap**

Read: `docs/architecture/80-framework-adapters.md`
Expected: the route set exists, but the developer-facing client namespace contract is not yet described clearly enough.

**Step 3: Update the doc minimally**

Add or refine sections so the doc explicitly states:

- the server runtime and fetch client share one API shape
- canonical route mapping for `list`, `get`, `create`, `update`, and `archive`
- action routes for workflow methods like `transition`
- Drizzle-inspired plain-object query inputs for `list` and `search`

**Step 4: Review the edited markdown**

Expected: the document aligns transport behavior with the approved client API without inventing extra endpoints.

**Step 5: Commit**

```bash
git add docs/architecture/80-framework-adapters.md
git commit -m "docs: map unified client methods to transport"
```

### Task 3: Update the getting started example to reflect the approved client API

**Files:**
- Modify: `docs/examples/getting-started.md`
- Reference: `docs/plans/2026-04-26-client-api-design.md`

**Step 1: Write the missing doc assertions**

Create a short checklist for this file:

- show the approved `commerce.domain.method(...)` shape
- include `list`/`search` query examples with `where` and `orderBy`
- distinguish CRUD-like methods from workflow methods

**Step 2: Review the current doc to verify the gap**

Read: `docs/examples/getting-started.md`
Expected: it explains config and HTTP mounting, but it does not yet show the intended developer-facing client usage clearly.

**Step 3: Update the example doc minimally**

Add a small example section that shows illustrative usage such as:

```ts
const product = await commerce.products.create({ ... })
const order = await commerce.orders.create({ ... })
const results = await commerce.products.list({
  where: { status: "active" },
  orderBy: [{ createdAt: "desc" }],
  limit: 20,
})
```

Optionally include a `search` example only if it is described as domain-specific rather than universal.

**Step 4: Review the edited markdown**

Expected: the example is clearly illustrative and consistent with the architecture docs.

**Step 5: Commit**

```bash
git add docs/examples/getting-started.md
git commit -m "docs: show approved client api examples"
```

### Task 4: Cross-check consistency across all updated docs

**Files:**
- Verify: `docs/architecture/60-type-inference.md`
- Verify: `docs/architecture/80-framework-adapters.md`
- Verify: `docs/examples/getting-started.md`
- Verify: `docs/plans/2026-04-26-client-api-design.md`

**Step 1: Review naming consistency**

Check that all docs use the same vocabulary for:

- `commerce.domain.method(...)`
- `list`, `get`, `create`, `update`, `archive`
- `transition` as a workflow method
- `where`, `orderBy`, `limit`, `cursor`

**Step 2: Review capability language**

Check that all docs consistently say unsupported methods are omitted at compile time rather than exposed and rejected later.

**Step 3: Review runtime language**

Check that all docs consistently describe one shared public API shape across in-process and HTTP-backed runtimes.

**Step 4: Final manual verification**

Expected: no doc contradicts the approved design and no package scaffolding or `package.json` creation is suggested in this phase.

**Step 5: Commit**

```bash
git add docs/architecture/60-type-inference.md docs/architecture/80-framework-adapters.md docs/examples/getting-started.md docs/plans/2026-04-26-client-api.md
git commit -m "docs: align client api documentation"
```
