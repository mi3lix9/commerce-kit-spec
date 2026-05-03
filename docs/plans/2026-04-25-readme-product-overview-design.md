# README and Product Overview Design

**Date:** 2026-04-25
**Status:** Approved

## Goal

Replace the root `bisuness-overview.md` file with a split documentation model where `README.md` becomes the repository entrypoint and `docs/product-overview.md` becomes the long-form product and business narrative.

## Decisions

### 1. File roles and locations

- `README.md` lives at the repository root and is the first document readers should see.
- `docs/product-overview.md` holds the fuller product/business narrative currently living in `bisuness-overview.md`.
- `docs/architecture/00-index.md` remains the canonical technical architecture entrypoint.

### 2. Content split

`README.md` should be concise and repository-facing:

- short project introduction
- short explanation of what Commerce Kit is
- brief why-it-exists summary
- current status
- documentation map
- short v1 scope summary
- principles summary
- short non-goals list
- next steps for the repository

`docs/product-overview.md` should hold the fuller narrative:

- positioning
- problem framing
- audience
- what Commerce Kit covers
- what it deliberately does not cover
- plugin ecosystem value
- single-vendor to multi-vendor story
- comparison framing and long-term value

### 3. Rewrite sequence

1. Create `README.md`
2. Rewrite `bisuness-overview.md` into `docs/product-overview.md`
3. Update doc links that still point at `/bisuness-overview.md`
4. Remove `bisuness-overview.md`

### 4. Writing rules

- `README.md` must stay concise and act as a repo landing page
- `docs/product-overview.md` can be longer and more narrative
- `docs/architecture/*` must stay technical and normative
- avoid duplicating architecture content in either README or product overview

### 5. Initial README shape

```md
# Commerce Kit

Short positioning statement.

## What is Commerce Kit?
## Why it exists
## Status
## Documentation
## Planned v1 scope
## Principles
## What’s not included
## Next
```

## Notes

The existing filename `bisuness-overview.md` contains a typo and should be removed as part of this rewrite rather than preserved.
