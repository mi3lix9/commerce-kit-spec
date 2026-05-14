# Pricing and Calculations

## Purpose

Define the monetary calculation model, pricing pipeline, rounding behavior, recalculation timing, and audit guarantees for Commerce Kit.

## Non-goals

- defining payment provider contracts
- repeating core entity definitions outside pricing-specific invariants
- defining plugin packaging or framework transport behavior

## Core decisions

- All calculations use integer minor units only.
- Tax is modeled as a pricing rule, not as a core adapter.
- Stored totals are authoritative on read.
- Existing orders are not retroactively recalculated when pricing rules change.
- Pricing execution must remain auditable.

## Money primitive

Public SDK and HTTP contracts represent money as a plain object with integer minor-unit amounts.

```ts
type Money = {
  amount: number
  currency: string
}
```

Normative rules:

- `amount` is always an integer in minor currency units.
- `amount` must be finite and a safe JavaScript integer at public API boundaries.
- `currency` is always an ISO 4217 code.
- Currency mismatches throw immediately.
- Internal calculation code may use `bigint`, but public HTTP/client contracts do not expose `bigint`.
- Display formatting belongs to optional client-side plugins or utilities, not to the core `Money` shape.

Internal calculation helpers may expose immutable money operations, but those helpers are implementation details unless a later RFC promotes them to public API.

### Money helpers

`Money` is exposed as a sealed shape with constructor and arithmetic helpers. Direct field arithmetic is discouraged because it lets currency mismatches compile.

```ts
import { money, add, sub, mul } from "commerce-kit/money"

const a = money(1500, "SAR")
const b = money(2000, "SAR")
const c = money(500, "USD")

add(a, b)   // ✅ Money { amount: 3500, currency: "SAR" }
add(a, c)   // ❌ compile error: currency literals differ ("SAR" vs "USD")
mul(a, 2)   // ✅ Money { amount: 3000, currency: "SAR" }
```

Each helper is generic over the currency literal so cross-currency arithmetic is rejected at compile time, not at runtime. Branded identifiers follow the same principle — see [60-type-inference.md](./60-type-inference.md#branded-identifiers).

## Rounding

Rounding is configured at `createCommerce()` time:

```ts
createCommerce({ rounding: "half_up" | "half_even" | "floor" | "ceil" })
```

Default: `half_up`.

The configured rounding strategy applies in every `multiply()` and `percentage()` operation.

## Calculation timing

Totals are recalculated only at these moments:

- order creation
- state transitions when totals are affected

Totals are not recalculated:

- when historical pricing rules are edited later
- when orders are read back from storage

## Pricing pipeline

The calculation pipeline is a named-step pipeline owned by the calculation engine. Pipelines are `string[]` of step IDs and can be set at app config time or stored per-merchant for runtime mutation. Step authoring, registration, resolution, and lifecycle live in [35-calculation-engine.md](./35-calculation-engine.md).

This document owns only the money invariants the pipeline must respect:

- Every step's emitted amounts are integer minor units.
- Steps may not reimplement rounding — the engine applies the configured rounding strategy in finalization.
- Allocation of order-level adjustments across line items is a fixed core algorithm and may not be overridden by steps.

The four canonical step kinds — `discount`, `serviceCharge`, `tax`, `fulfillment` — are recognized for derived totals (`discountTotal`, `serviceChargeTotal`, etc.). Steps with a custom `kind` participate in the order's `adjustments` audit list but do not contribute to a canonical total column.

## Audit guarantees

- Every calculation produces an `adjustments: Adjustment[]` audit record persisted on `order`. This is the canonical source of truth for how a total was derived.
- Each `Adjustment` records its `source` (the step ID that emitted it), `kind`, `amount`, and `appliesTo`.
- Derived totals (`subtotal`, `discountTotal`, `serviceChargeTotal`, `taxTotal`, `fulfillmentTotal`, `total`) are stored as columns for query convenience, but the adjustments array is the authoritative trace.
- Rule configuration is snapshotted into adjustment `metadata` at the time the step runs, so historical orders remain interpretable after rule edits.

## Cross-links

- Calculation engine, named steps, runtime overrides: [35-calculation-engine.md](./35-calculation-engine.md)
- Core engine boundaries: [10-core-engine.md](./10-core-engine.md)
- Persistence invariants: [20-data-model.md](./20-data-model.md)
- Plugin-contributed steps: [40-plugin-system.md](./40-plugin-system.md)

## Future RFCs

- Multi-currency conversion policy beyond the current `Money` contract
- Adjustment indexing for reporting workloads
