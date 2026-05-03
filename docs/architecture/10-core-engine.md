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
3. **Complexity is opt-in** — minimal commerce setups should not carry the cost of optional capabilities.
4. **Plugin-first feature model** — features should ship as plugins before being considered core.
5. **Never wrong silently** — invalid transitions, oversold inventory, and float arithmetic on money must fail clearly.

## Core scope

Core v1 always includes:

- products and variants
- orders and the order lifecycle
- the payment adapter interface

Core v1 assumes a simple-store baseline. Marketplace capabilities such as vendor ownership, vendor operations, and order splitting are not core responsibilities; they are installed through plugins.

Core also bundles optional adapter interfaces for fulfillment and payout providers, but they are dormant until at least one provider is registered. Fulfillment is the provider-neutral umbrella for carrier shipping, local delivery, pickup, and digital delivery.

## Lifecycle and boundaries

Core owns the commerce engine itself:

- catalog primitives
- order creation and status transitions
- payment orchestration and payment event recording
- calculation execution and stored totals
- auditability for order transitions

Marketplace-specific domain expansion belongs outside core, including:

- vendor identity and lifecycle
- product ownership by vendor
- vendor-facing APIs and the route contributions that expose them through framework adapters
- order splitting and per-vendor fulfillment coordination
- payouts, commissions, and onboarding workflows

Core does not own provider-specific payment or fulfillment implementations. Those are supplied through adapters or plugins against documented contracts.

Optional adapter interfaces follow a dormant activation pattern:

- if no provider is registered, Commerce Kit creates no related schema, states, routes, hooks, or typed API surface
- when a provider is registered, `createCommerce()` activates the interface and materializes the related schema and API

Core defines the adapter contract for payout providers, but it does not define marketplace payout policy. A marketplace plugin may depend on the payout adapter interface and supply the vendor-facing payout, commission, and onboarding workflow that uses it.

Tax is not a core adapter. Tax is modeled as a pricing rule plugin, and zero tax is a valid configuration.

## Money and calculation boundaries

Core owns the existence of the `Money` primitive and the requirement that calculations use integer minor units only.

The detailed pricing pipeline, rounding behavior, recalculation timing, and audit guarantees are defined in [30-pricing-and-calculations.md](./30-pricing-and-calculations.md).

## Future RFCs

- Changes to core-engine scope boundaries
- New always-on core responsibilities that cannot be modeled as plugins or optional adapters
