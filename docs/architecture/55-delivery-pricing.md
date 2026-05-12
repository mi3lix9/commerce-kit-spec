# Delivery Pricing Strategies

## Purpose

Define how delivery fees are computed: the `deliveryPricing: []` config slot, the `DeliveryPricingStrategy` interface, the built-in strategies that ship with Commerce Kit, and the `DistanceProvider` abstraction that strategies use for geolocation work.

Strategies are declared at app boot via `deliveryPricing: [...]` and passed inline at method-create time via typed factories. Boot-time declaration lets core validate every DB-stored method's strategy tag at startup and surface the registry to admin UIs via `commerce.delivery.strategies.list()`.

Strategies are orthogonal to delivery adapters ([54-delivery-adapters.md](./54-delivery-adapters.md)). An adapter handles dispatch; a strategy computes the fee. Each delivery method picks one of each, independently.

## Non-goals

- Defining the `DeliveryAdapter` contract or any dispatch concerns — see [54-delivery-adapters.md](./54-delivery-adapters.md)
- Specific algorithms baked into core — core ships no Google Maps integration, no distance math, no zone resolver. Strategies are plugin / package territory.
- Pricing for the `fulfillment` or `shipping` slots — those use their own concepts (in-house types and carrier rates respectively)

## Core decisions

- Strategies are exposed to apps as **typed factory functions** — `distanceBased({...})`, `flat({...})`, etc. — used inline at method-create time. The factory carries both the strategy tag and the typed settings.
- A method's `pricing` field is a discriminated union of factory outputs. TS infers settings shape from the chosen factory, so passing wrong settings is a compile error, not a `validateSettings` runtime throw.
- Strategies are declared at app boot via the `deliveryPricing: []` config slot. Built-in and custom strategies use the same declaration mechanism — both go in the array.
- Boot-time declaration enables startup validation: if a DB-stored method references a tag not in the registry, `createCommerce()` throws a concrete error instead of failing at calc time.
- Core dispatches to the strategy at calculation time via the built-in `core:delivery-fee` step, looking up the strategy by the `strategy` tag stored on the method.
- Core does not ship any specific algorithm. The built-in strategies that DO ship live in `@commerce-kit/delivery-pricing-strategies`, a separate package.
- The `DistanceProvider` abstraction lets strategies fetch distance/duration without depending on a specific mapping service. Provider implementations (Google Maps, Mapbox, haversine) ship as sub-modules of `@commerce-kit/delivery-pricing-strategies`.

## Declaring strategies

Strategies are declared once at boot. The `deliveryPricing: []` slot accepts both built-in and custom strategies through the same shape:

```ts
import {
  distanceBasedStrategy,
  flatStrategy,
  freeStrategy,
} from '@commerce-kit/delivery-pricing-strategies'

createCommerce({
  ...,
  delivery: [leajlakDelivery({ apiKey: env.LEAJLAK })],
  deliveryPricing: [
    distanceBasedStrategy(),
    flatStrategy(),
    freeStrategy(),
  ],
})
```

At method-create time, the matching factory carries the typed settings:

```ts
import { distanceBased, flat } from '@commerce-kit/delivery-pricing-strategies'

await commerce.delivery.methods.create({
  adapter: 'leajlak',
  name: 'Riyadh Express',
  pricing: distanceBased({
    basePrice: money(1700, 'SAR'),
    baseDistanceMeters: 3000,
    perKm: money(100, 'SAR'),
  }),
})
```

Each strategy package exports two paired items: the `*Strategy()` registration (for the config slot) and the lower-case factory (for the call site). They share a tag string that core uses to link the call site to its calculator.

Strategies that need a `DistanceProvider` accept it via dependency injection at the factory call site:

```ts
import { distanceBased } from '@commerce-kit/delivery-pricing-strategies'
import { googleMaps } from '@commerce-kit/delivery-pricing-strategies/providers/google-maps'

const distance = googleMaps({ apiKey: env.GOOGLE_KEY })

await commerce.delivery.methods.create({
  adapter: 'leajlak',
  pricing: distanceBased({ ...settings, distanceProvider: distance }),
})
```

Apps that always use the same distance provider typically wrap this in a small helper. Core does not provide one — it's a one-liner and apps usually want control over which provider runs where.

## Custom strategies

Custom strategies use the same shape as built-ins — register at boot, use at call site:

```ts
import { vipTierStrategy, vipTier } from './pricing/vip-tier'

createCommerce({
  ...,
  deliveryPricing: [
    distanceBasedStrategy(),
    vipTierStrategy(),               // custom
  ],
})

await commerce.delivery.methods.create({
  adapter: 'leajlak',
  pricing: vipTier({ tier: 'gold' }),
})
```

Strategy tags use the `<scope>:<name>` convention (`app:`, `<plugin-id>:`) so they don't collide with the unprefixed built-ins. See [Authoring a custom strategy](#authoring-a-custom-strategy) below for the strategy author's surface.

## `DeliveryPricingStrategy` interface

A strategy author defines both the factory (input shape, type tag) and the calculator (runtime behavior). Core ties them together via the discriminated union.

```ts
interface DeliveryPricingStrategy<TSettings, TTag extends string> {
  tag: TTag                                       // strategy ID, e.g. 'distance-based'
  validateSettings(settings: TSettings): void     // defense in depth at method-create
  calculate(ctx: StrategyContext<TSettings>): Promise<CalculateResult>
}

type StrategyContext<TSettings> = {
  method: DeliveryMethod
  settings: TSettings              // already typed — no jsonb cast needed
  items: PricingItem[]
  subtotal: Money
  deliveryAddress: Address
  branchAddress?: Address          // origin (present when tenancy.branches is on)
  request: RequestContext
}

type CalculateResult = {
  amount: Money
  metadata?: Record<string, unknown>   // distance, duration, zone, provider quote ID, etc.
}
```

The factory returns a tagged settings object that's the value stored in `deliveryMethod.pricing`:

```ts
type PricingValue<TSettings, TTag extends string> = { strategy: TTag } & TSettings

// Built-in factories are just:
function distanceBased(settings: DistanceBasedSettings): PricingValue<DistanceBasedSettings, 'distance-based'> {
  return { strategy: 'distance-based', ...settings }
}
```

The discriminated union of all strategies in `deliveryPricing: []` becomes the type of `deliveryMethod.pricing` inputs. TS picks the right `TSettings` from the `strategy` tag.

Strategies are pure functions of context. They do not perform side effects beyond reading their own configured dependencies (a distance provider, a remote quote API). They are not allowed to mutate the order or emit other adjustments — that's the calculation engine's job.

### Throwing

Strategies throw `CommerceValidationError` for unservable contexts:

- `below_min_amount` — `subtotal < method.minOrderAmount`
- `out_of_range` — computed distance exceeds `method.maxDistanceMeters`
- `no_zone_match` — address doesn't fall in any configured zone
- `provider_unavailable` — live quote API failed or timed out
- `geocoding_failed` — distance provider couldn't resolve coordinates

Errors propagate to the calling operation (`orders.calculate` / `orders.checkout`). The customer sees a structured error.

## Built-in strategies

These ship in `@commerce-kit/delivery-pricing-strategies`. Apps install the package and reference the strategies they want.

### `free()`

Always returns zero. No settings.

```ts
pricing: free()
```

Useful when a merchant wants free delivery above a `minOrderAmount` (combine with the min-order guard).

### `flat({ amount })`

Returns a fixed amount per delivery.

```ts
pricing: flat({ amount: money(2500, 'SAR') })
```

### `distanceBased({ ... })`

Jaicome's pricing model. Settings shape:

```ts
pricing: distanceBased({
  basePrice: money(1700, 'SAR'),    // flat fee within baseDistance
  baseDistanceMeters: 3000,         // flat-fee radius
  perKm: money(100, 'SAR'),         // additional fee per km beyond baseDistance
  perMin: money(50, 'SAR'),         // optional time-based addition
  roundToNearest: 100,              // default 100 (nearest SAR)
  distanceProvider,                 // DistanceProvider, injected
})
```

Algorithm:

```
{ distanceMeters, durationSeconds } = await distanceProvider({ origin, destination })

extraDistanceM = max(0, distanceMeters - baseDistanceMeters)
extraDurationS = max(0, durationSeconds - baseDurationSeconds)

fee = basePrice
    + extraDistanceM * (perKm / 1000)
    + (perMin ? extraDurationS * (perMin / 60) : 0)

fee = round(fee, roundToNearest)
```

Throws `out_of_range` if `method.maxDistanceMeters` is set and distance exceeds it.

### `tieredDistance({ tiers, distanceProvider })`

Distance brackets — different fee per bracket.

```ts
pricing: tieredDistance({
  tiers: [
    { maxDistanceMeters: 5000, amount: money(1500, 'SAR') },
    { maxDistanceMeters: 15000, amount: money(3000, 'SAR') },
  ],
  distanceProvider,
})
```

Picks the first tier whose `maxDistanceMeters` is not exceeded by the resolved distance. Throws `out_of_range` if no tier matches.

### `weight({ tiers })`

Weight-based brackets. Reads each item's weight from product metadata.

```ts
pricing: weight({
  tiers: [
    { maxWeightG: 1000, amount: money(1000, 'SAR') },
    { maxWeightG: 5000, amount: money(2500, 'SAR') },
  ],
})
```

Sums item weights × quantities, picks the first tier whose `maxWeightG` is not exceeded.

### `zone({ zones })`

Zone lookup based on address.

```ts
pricing: zone({
  zones: [
    { id: 'central', match: { kind: 'city', cities: ['Riyadh'] }, amount: money(1500, 'SAR') },
    { id: 'outer', match: { kind: 'postal-code', codes: ['12345'] }, amount: money(3000, 'SAR') },
  ],
})

type ZoneMatcher =
  | { kind: 'postal-code'; codes: string[] }
  | { kind: 'city'; cities: string[] }
  | { kind: 'country'; countries: string[] }
  | { kind: 'polygon'; polygon: Coordinates[] }
```

Picks the first matching zone. Throws `no_zone_match` if no zone matches the delivery address.

### Authoring a custom strategy

A strategy package exports two paired items: the registration (for `deliveryPricing: []`) and the call-site factory:

```ts
// strategy author's package
export function vipTierStrategy(): DeliveryPricingStrategy<VipTierSettings, 'app:vip-tier'> {
  return {
    tag: 'app:vip-tier',
    validateSettings: (s) => { /* throw on misconfig */ },
    calculate: async ({ settings, subtotal }) => {
      return { amount: /* custom math */ }
    },
  }
}

export function vipTier(settings: VipTierSettings): PricingValue<VipTierSettings, 'app:vip-tier'> {
  return { strategy: 'app:vip-tier', ...settings }
}
```

App wires it in like any built-in:

```ts
createCommerce({
  deliveryPricing: [vipTierStrategy()],
})

// then at method-create:
pricing: vipTier({ tier: 'gold' })
```

The two are kept paired by their shared tag string; core's type machinery links the factory's discriminant to the registered calculator.

## `DistanceProvider` interface

Strategies that need distance/duration use the `DistanceProvider` abstraction:

```ts
type DistanceProvider = (input: {
  origin: Coordinates
  destination: Coordinates
}) => Promise<{
  distanceMeters: number
  durationSeconds: number
}>
```

Implementations live in `@commerce-kit/delivery-pricing-strategies/providers/*`:

| Sub-module | Provider |
|---|---|
| `/providers/google-maps` | Google Maps Distance Matrix API |
| `/providers/mapbox` | Mapbox Matrix API |
| `/providers/haversine` | Pure-math haversine formula, no external API |

Strategies that need distance receive a provider at the factory call site:

```ts
distanceBased({
  ...settings,
  distanceProvider: googleMaps({ apiKey: env.GOOGLE_KEY }),
})
```

Different strategies can use different providers in the same app. A merchant in a remote area without Google Maps coverage could use a haversine fallback while urban merchants use Google Maps.

Distance providers handle their own caching internally if needed. Core does not provide a caching layer.

## How a method picks a strategy

At method creation, the merchant (or admin acting on their behalf) picks an adapter and constructs pricing inline with a factory call:

```ts
await commerce.delivery.methods.create({
  adapter: 'leajlak',
  name: 'Riyadh Delivery',
  branch: 'br_riyadh',
  pricing: distanceBased({
    basePrice: money(1700, 'SAR'),
    baseDistanceMeters: 3000,
    perKm: money(100, 'SAR'),
    distanceProvider,
  }),
  minOrderAmount: money(5000, 'SAR'),
  maxDistanceMeters: 20000,
})
```

The same merchant can create another method with the same adapter and a different strategy:

```ts
await commerce.delivery.methods.create({
  adapter: 'leajlak',
  name: 'Jeddah Delivery',
  branch: 'br_jeddah',
  pricing: flat({ amount: money(2500, 'SAR') }),
})
```

Per-merchant pricing decisions live at the method level. The platform doesn't dictate which strategy a merchant uses.

## Validation

At `createCommerce()` startup:
- Each entry in `deliveryPricing: []` must implement `tag`, `validateSettings`, and `calculate`.
- Strategy tags must be unique within the array.
- Every existing DB-stored `deliveryMethod.pricing.strategy` value must reference a tag in the array — startup throws otherwise. This catches "I removed the strategy import" before it hits production.

At `deliveryMethod.create` / `update`:
- TS rejects malformed `pricing` at compile time via the discriminated union.
- Runtime `validateSettings` runs as defense-in-depth, mainly for dynamically constructed inputs (admin forms, API payloads). Misconfiguration throws `CommerceValidationError`.

At `orders.calculate` / `orders.checkout`:
- The order's `deliveryMethodId` resolves to a method.
- Core looks up the strategy by `method.pricing.strategy`, then calls `calculate`. Throwing aborts the operation.

## Calculation engine integration

When the `delivery` slot is active, core registers a built-in calculation step `core:delivery-fee` that:

1. Reads the order's `deliveryMethodId`. If null, emits nothing.
2. Loads the method, reads `method.pricing.strategy` to look up the registered strategy.
3. Builds `StrategyContext` from the calculation state + `method.pricing` settings + order's `deliveryAddress` + branch coordinates.
4. Enforces the method's `minOrderAmount` guard (throws if violated).
5. Calls `strategy.calculate(ctx)`.
6. Emits a fulfillment adjustment with the result.

The step is part of the default pipeline. Apps can replace it via the calculation engine's standard mechanisms — see [35-calculation-engine.md](./35-calculation-engine.md).

## Server SDK

When the `delivery` slot is active, the union of built-in + custom strategies is queryable for admin UIs:

```ts
commerce.delivery.strategies.list(): Promise<Array<{
  tag: string                          // 'distance-based' | 'flat' | 'app:vip-tier' | ...
  description: string | null
  settingsSchema: JsonSchema           // serialized Zod schema
}>>
```

This is the only read against the strategy registry. It supports a "create delivery method" form that needs to render the right settings inputs for the chosen strategy.

## Why strategies are separate from adapters

Splitting dispatch (adapter) from pricing (strategy) means:

- A merchant using Leajlak for dispatch can choose any pricing model they want — they aren't forced into Leajlak's quote.
- A merchant with 4 delivery providers can use unified pricing across all of them. Pick the same strategy for every method; route dispatch via the adapter.
- Provider-supplied pricing (e.g., Leajlak's live quote API) is just another strategy. It can be picked or ignored independently of using the same adapter for dispatch.

The delivery adapter registry (`delivery: [...]`) and the import-time strategy registry are deliberately independent. They compose at the method level.

## Tenancy

Strategies themselves are app-level. They are not per-merchant.

What IS per-merchant is the **selection** of strategy at the method level. Each merchant's `deliveryMethod` rows declare which strategy applies. Different merchants on the same app can use different strategies for their methods.

This means platform operators choose which strategies are available to merchants by declaring them in `deliveryPricing: []`. Adding a new strategy is an app-level change, not a per-merchant change.

## Packaging

| Package | Owns |
|---|---|
| `commerce-kit` (core) | `DeliveryPricingStrategy` interface, the `deliveryPricing: []` config slot, the strategy registry, the `core:delivery-fee` calculation step, startup validation |
| `@commerce-kit/delivery-pricing-strategies` | Built-in registrations + factories (`free`/`freeStrategy`, `flat`/`flatStrategy`, `distanceBased`/`distanceBasedStrategy`, etc.) + `DistanceProvider` interface + ready-made distance providers as sub-modules |

Core ships zero algorithms. Even the trivial `free` strategy lives in the strategies package — uniform contract, no special-casing.

Strategies the app doesn't list in `deliveryPricing: []` are never in the registry, and tree-shaking drops them. Apps that only use `distanceBased` ship only `distanceBased`.

## Cross-links

- Delivery adapter contract: [54-delivery-adapters.md](./54-delivery-adapters.md)
- Calculation engine and `core:delivery-fee` step: [35-calculation-engine.md](./35-calculation-engine.md)
- Tenancy and per-branch methods: [12-tenancy.md](./12-tenancy.md)
- Money primitives and rounding: [30-pricing-and-calculations.md](./30-pricing-and-calculations.md)
- Type inference for the `pricingStrategyId` literal union: [60-type-inference.md](./60-type-inference.md)

## Future RFCs

- Strategy composition (e.g., "apply distance-based, then add a flat surcharge")
- Conditional strategies that pick another strategy based on subtotal or distance
- Per-merchant runtime strategy overrides (analogous to `calculation.runtime: true`)
- Strategy versioning for safe evolution of settings schemas
