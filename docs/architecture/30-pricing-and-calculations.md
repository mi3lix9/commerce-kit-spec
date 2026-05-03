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

The calculation pipeline is:

1. Sum line items into subtotal
2. Apply `discount` rules to line items/order amounts
3. Add fulfillment charges from the selected fulfillment method
4. Apply pre-tax `service_charge` and `custom` rules
5. Apply `tax` rules to the configured taxable base
6. Apply post-tax `service_charge` and `custom` rules
7. Persist final total

Fulfillment-charge rules:

- fulfillment fees enter the pipeline after discounts by default
- fulfillment charges are not discountable unless an installed pricing rule explicitly targets them
- fulfillment charges are taxable only when the active tax rules include them in the taxable base
- `fulfillmentTotal` remains persisted as an order field when fulfillment participates in the pricing pipeline

Tax defaults to the discounted taxable base. A rule may explicitly choose the gross pre-discount taxable base only for jurisdictions that require tax before discounts are applied. In this document, `pre_tax` means "before tax is applied in the pipeline," not "before discounts necessarily run."

## Audit guarantees

- Every calculation step is auditable through `pricingRuleExecution` records when a pricing plugin or deployment enables that persistence surface.
- Rule configuration is snapshotted at execution time.
- Historical orders keep the totals and rule snapshots produced at the time of execution.

`pricingRuleExecution` is not part of the minimal core entity list in `20-data-model.md`. Its ownership boundary is the pricing system and any installed pricing-rule plugins that choose to persist execution traces. Core owns the audit requirement for applied totals and snapshots; the detailed execution-record persistence surface belongs to the pricing boundary rather than to unrelated domains.

## Cross-links

- Core engine boundaries live in [10-core-engine.md](./10-core-engine.md)
- Persistence invariants live in [20-data-model.md](./20-data-model.md)
- Plugin-owned pricing extensions belong in [40-plugin-system.md](./40-plugin-system.md)

## Future RFCs

- Multi-currency conversion policy beyond the current `Money` contract
- Additional calculation audit surfaces beyond `pricingRuleExecution`
