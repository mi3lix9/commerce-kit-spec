# Core Engine

## Purpose

Define what the Commerce Kit core engine owns, what it excludes, and the rules that govern its operation.

## Non-goals

Core does not own:

- storefront UI
- email and notifications
- CMS and content
- admin dashboard
- authentication
- analytics and reporting

These remain outside the commerce engine under the guest-in-your-stack rule.

## Core principles

1. **Library, not platform** — Commerce Kit integrates into an existing stack instead of replacing the runtime, ORM, deployment model, or application conventions.
2. **Application-owned environment** — the consuming application keeps its framework, auth, database, and deployment pipeline.
3. **Complexity is opt-in** — minimal commerce setups should not carry the cost of optional capabilities. Tenancy, fulfillment, and other advanced features are dormant until declared.
4. **Plugin-first feature model** — features ship as plugins before being considered core. Tenancy is the exception: it is core because plugins can't safely reinvent the ownership model.
5. **Never wrong silently** — invalid transitions, oversold inventory, and float arithmetic on money must fail clearly.
6. **Plugins are tenancy-agnostic** — the same plugin code runs in simple-store, multi-merchant, and multi-branch modes.

## Core scope

Core v1 always includes:

- products and variants
- orders and the order lifecycle
- the payment adapter interface
- customers
- the hook system (keyed `on` handlers)
- the plugin system (`plugin()`, `table()`, column helpers)
- the tenancy contract (column helpers, `support` block, hierarchical reads)

Core v1 assumes a simple-store baseline. Capabilities like multi-merchant ownership, multi-branch operation, marketplace checkout, fulfillment providers, and scheduled tasks are dormant unless declared in `createCommerce()`.

## Dormant activation

Core uses dormant activation uniformly across optional capabilities:

| Capability | Activated by | When dormant |
|---|---|---|
| Tenancy (`merchants`) | `tenancy: { merchants: true }` | no `merchants` table, no `merchant()` column injection |
| Tenancy (`branches`) | `tenancy: { branches: true }` | no `branches` table, no `branch()` column injection |
| Marketplace checkout | `tenancy: { checkout: 'split' }` | no `orderGroup` table, single-merchant carts enforced |
| Server cart persistence | `cart: { ... }` adapter config | no cart table, `commerce.cart` namespace absent |
| Fulfillment | at least one fulfillment adapter | no fulfillment schema, no `commerce.fulfillment.*` |
| Payout | at least one payout adapter | no payout schema, no related SDK surface |
| Scheduler / deferred tasks | scheduler adapter | no deferred task scheduling; `commerce.tasks.run` still works |
| Storage | storage adapter | no upload primitives |

When dormant, there are zero schema rows, zero columns, and no related typed SDK surface. The simple-store DX is unchanged regardless of which capabilities exist in core.

## Lifecycle and boundaries

Core owns:

- catalog primitives (products, variants)
- order creation and status transitions
- payment orchestration and append-only payment ledger
- calculation execution and stored totals
- auditability for order transitions
- the tenancy contract (merchant/branch primitives when active)
- the marketplace order-splitting mechanic (when `tenancy.checkout: 'split'`)
- the plugin contract and plugin-owned data access path

Core does not own:

- provider-specific payment or fulfillment implementations — supplied via adapters
- commission policy, payout scheduling, vendor approval workflows — application-owned
- modifiers, coupons, business hours, tax computation strategies — plugin-owned
- storefront, admin UI, notifications, analytics — application-owned

## Tax

Tax is not a core adapter. Tax is modeled as a pricing rule plugin, and zero tax is a valid configuration.

## Money and calculation boundaries

Core owns the existence of the `Money` primitive and the requirement that calculations use integer minor units only.

The detailed pricing pipeline, rounding behavior, recalculation timing, and audit guarantees are defined in [30-pricing-and-calculations.md](./30-pricing-and-calculations.md).

## Why tenancy is in core, not a plugin

Earlier drafts proposed `@commerce-kit/branches` and `@commerce-kit/marketplace` as plugins. That design was rejected because:

- "Who owns this row" and "where does it operate" are foundational data-model concerns. Plugins that handle them would need migrations every time a row changes ownership.
- Other plugins (coupons, business hours, modifiers) would need to depend on the branches/marketplace plugins to access scoping, producing plugin-on-plugin dependency chains and version conflicts.
- The `merchant` / `branch` columns and their hierarchical resolution must apply uniformly to every table in the system — plugin and core alike. A plugin cannot enforce that.

Tenancy is therefore a core capability with dormant activation. Plugins declare their tenancy `support` and use `merchant()` / `branch()` column helpers, but they never reinvent the model. See [12-tenancy.md](./12-tenancy.md).

## Future RFCs

- Changes to core-engine scope boundaries
- New always-on core responsibilities that cannot be modeled as plugins or optional adapters
- Tenancy axes beyond merchants and branches
