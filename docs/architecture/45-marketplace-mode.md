# Marketplace Mode

## Purpose

Define the marketplace checkout mode: a configuration of `tenancy.merchants` that allows customers to assemble carts spanning multiple merchants and have the resulting order split per merchant at checkout.

Marketplace is **not a separate plugin**. It is a checkout strategy that activates when `tenancy.merchants` is enabled and the application opts into the cross-merchant cart model.

## Non-goals

- Vendor onboarding workflows (KYC, contracts) — application-owned
- Commission policy and rate negotiation — application-owned
- Payout provider implementation — adapter concern, see [50-adapter-system.md](./50-adapter-system.md)
- Storefront browsing UX — application-owned

## Core decisions

- Marketplace mode is a setting on `tenancy.merchants`, not a separate plugin.
- Two checkout strategies exist when `tenancy.merchants: true`:
  - `single-merchant` (default) — each cart belongs to exactly one merchant; cross-merchant carts are rejected.
  - `split` — carts may span merchants; checkout produces one `order` per merchant, with a parent linking record.
- Core owns the order-splitting mechanics. Application logic (commission, payout timing, vendor approval) lives outside core.
- A single core `order` always belongs to exactly one merchant. The "cross-merchant order" concept lives as a parent group of single-merchant orders.

## Config

```ts
createCommerce({
  ...,
  tenancy: {
    merchants: true,
    branches: false,
    checkout: 'split',     // marketplace mode
  },
})
```

| `tenancy.checkout` | Behavior |
|---|---|
| `'single-merchant'` (default) | Each cart has one merchant. Items from different merchants in the same cart fail at `orders.calculate` and `orders.checkout`. |
| `'split'` | Cart may span merchants. Checkout produces N orders + 1 group record. |

`checkout: 'split'` is only valid when `tenancy.merchants: true`. Otherwise it's a config error.

## Order group

When `checkout: 'split'` is on, core materializes an `orderGroup` table:

```ts
orderGroup = {
  id: string
  customerId: string | null
  status: 'pending' | 'partial' | 'complete' | 'failed'
  currency: string
  total: Money              // sum of child orders
  createdAt: Date
  updatedAt: Date
}

order = {
  ...existing fields,
  merchantId: string         // required when tenancy.merchants is on
  groupId: string | null     // present when this order is part of a split
}
```

`order.groupId` is null in `single-merchant` mode and points to the parent group in `split` mode.

## Checkout flow in split mode

`orders.checkout` accepts a cart with items potentially spanning merchants. Internally it:

1. Groups items by `product.merchant`.
2. Creates one `orderGroup` row.
3. Creates one `order` row per merchant group, each linked to the `orderGroup`.
4. Computes per-order totals using the same pricing pipeline as single-merchant checkout.
5. Initiates payment **once** for the group total (the payment adapter sees one transaction).
6. Records the payment against the group; child orders inherit settlement status.

```ts
const result = await commerce.orders.checkout({
  items: [
    { variantId: 'v_pizza_marg', quantity: 1 },     // merchant 'm1'
    { variantId: 'v_burger_classic', quantity: 2 }, // merchant 'm2'
  ],
  customerId: 'cust_1',
  payment: { adapterId: 'moyasar' },
})

// result.group.id          → the orderGroup
// result.orders            → array of one order per merchant
// result.paymentReference  → single payment for the group total
// result.paymentUrl        → single redirect URL when applicable
```

## Single-merchant mode (default)

In `single-merchant` mode (the default for `tenancy.merchants: true`), every cart is implicitly scoped to one merchant. Items from different merchants in the same cart fail validation at `orders.calculate` and `orders.checkout` with a concrete error:

```
✗ Cannot calculate order: items belong to multiple merchants (m1, m2).
  Marketplace mode is disabled. Set tenancy.checkout = 'split' to allow
  cross-merchant carts, or filter items to a single merchant before checkout.
```

This is the right default for SaaS-style multi-merchant deployments (Jaicome-style) where each merchant has its own storefront and customers never assemble cross-merchant carts.

## Per-merchant lifecycle

Each child order in a split group has its own lifecycle. Different child orders may be at different valid statuses simultaneously (one merchant fulfills before another).

- `order.status` for each child follows the standard state machine.
- `orderGroup.status` is derived from the child statuses:
  - `pending` — all children in early states (`draft`, `placed`)
  - `partial` — some children advanced past `placed`, not all complete or terminal
  - `complete` — all children in `completed`
  - `failed` — all children in `cancelled` / `failed`

Derivation is deterministic and applied automatically when child orders transition.

## Payment allocation

Payment is taken at the group level — the customer sees one charge. The internal payment ledger records the captured amount against the group. Refunds may be:

- **Per child order** — refund a single merchant's portion. The payment ledger records a partial refund.
- **Whole group** — refund the entire group. All child orders transition to `refunded` together.

## Commission and payouts

Core does not define commission policy. Applications layer it on via:

- A plugin that hooks `orders:checkout:after` and records commission per child order.
- The payout adapter, which executes settlement to merchant bank accounts based on application-defined commission records.

This keeps the marketplace plumbing in core (split, group, lifecycle) while business policy stays with the application.

## What about vendor identity, onboarding, catalog management?

These are not Commerce Kit concerns in marketplace mode:

- **Vendor identity** is the `merchant` entity itself ([12-tenancy.md](./12-tenancy.md)).
- **Onboarding** (KYC, contracts, banking) is application UX.
- **Vendor catalog management** is normal `commerce.products.*` calls with the merchant's context bound.
- **Vendor approval** is a status field on the application's own admin layer, not in core.

The marketplace plugin from earlier drafts has been collapsed into:

1. `tenancy.merchants: true` for the merchant primitive ([12-tenancy.md](./12-tenancy.md))
2. `tenancy.checkout: 'split'` for cross-merchant carts (this doc)
3. Application code for everything else (onboarding, commission, vendor UX)

This removes the plugin-on-plugin dependency hell of the previous design.

## Migration from single-merchant to split

Switching `tenancy.checkout` from `'single-merchant'` to `'split'` is a config change. It does not require historical data migration because:

- Existing orders all have `groupId: null` — they remain interpretable as single-merchant orders.
- New checkouts may produce groups; old orders are untouched.
- The `orderGroup` table is created automatically on first activation.

Switching from `'split'` back to `'single-merchant'` is rejected at config validation if any `order.groupId` is non-null — the data shape would become inconsistent.

## Cross-links

- Tenancy contract: [12-tenancy.md](./12-tenancy.md)
- Core data model and entity participation in tenancy: [20-data-model.md](./20-data-model.md)
- Pricing pipeline used per child order: [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Payout adapter interface: [50-adapter-system.md](./50-adapter-system.md)
- Server SDK changes when tenancy is on: [25-server-sdk.md](./25-server-sdk.md)

## Future RFCs

- Per-merchant fulfillment timing (each child order advances independently)
- Per-child refund flows with adapter support for partial captures
- Commission as a first-class core concept rather than application-owned
