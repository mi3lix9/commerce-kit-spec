# Docs Examples Design

**Date:** 2026-04-25
**Status:** Approved

## Goal

Add a small proposed-usage code example to `README.md` and a fuller starter example guide under `docs/examples/` so readers can understand the intended Commerce Kit API shape before implementation exists.

## Decisions

### 1. Example strategy

- `README.md` gets a short **Quick example** section.
- `docs/examples/getting-started.md` becomes the fuller starter example.
- The examples should optimize for a **realistic starter app** rather than a minimal or marketplace-only example.

### 2. Example content

The examples should show the full surface of `createCommerce(...)` at a high level:

- database adapter
- payment adapters
- shipping adapters
- delivery adapters
- plugins
- mode

The README example stays compact. The fuller guide shows:

1. proposed app structure
2. proposed `lib/commerce.ts`
3. framework mounting example
4. example request flow
5. explanation of what each adapter/plugin slot is for
6. note on dormant and optional adapters

### 3. Example tone and safety

- Present examples as **proposed v1 usage shape**
- Keep them concrete enough to teach the mental model
- Avoid presenting them as already shipped/runnable packages
- Keep naming aligned with the current architecture docs

### 4. File placement

- `README.md` — add `## Quick example`
- `docs/examples/getting-started.md` — full starter guide

### 5. Documentation roles

- `README.md` example answers: “what does using Commerce Kit look like?”
- `docs/examples/getting-started.md` answers: “how would I wire this into a real app?”
- architecture docs remain the canonical explanation of why the system is designed that way
