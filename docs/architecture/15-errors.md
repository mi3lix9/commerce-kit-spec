# Errors

## Purpose

Define the Commerce Kit error model: the error class hierarchy, core error codes, HTTP response shape, server-side throwing behavior, client SDK result pattern, and plugin error conventions.

## Non-goals

- Framework-specific UI error handling (React, etc.)
- Validation schema internals — only the output shape is defined here
- Repeating HTTP status mappings already listed in [80-framework-adapters.md](./80-framework-adapters.md)

## Core decisions

- All Commerce Kit errors extend a single `CommerceError` base class.
- Server-side operations (`commerce.*`) throw on failure.
- Client SDK operations (`client.*`) never throw — they always return `{ data, error }`.
- `error` in the client result is always `CommerceError | null`, never `unknown`.
- Framework wrappers (e.g. `@commerce-kit/react`) add `isLoading` on top of the base result shape.
- Plugin errors are namespaced with the plugin id to avoid code collisions with core.

---

## Error class hierarchy

```ts
// Base — usable directly or as a base for custom errors in plugins/hooks
class CommerceError extends Error {
  readonly code: string     // machine-readable, e.g. 'ORDER_NOT_FOUND'
  readonly status: number   // HTTP status code
  readonly data?: unknown   // optional structured context
}

class CommerceNotFoundError     extends CommerceError  // 404
class CommerceValidationError   extends CommerceError  // 400 — adds issues[]
class CommerceUnauthorizedError extends CommerceError  // 401
class CommerceForbiddenError    extends CommerceError  // 403
class CommerceOrderError        extends CommerceError  // 422
class CommerceStateError        extends CommerceError  // 422
class CommercePaymentError      extends CommerceError  // 402
class MoneyCurrencyError        extends CommerceError  // 422
class CommerceConfigError       extends CommerceError  // 500
class CommercePluginError       extends CommerceError  // 500
```

`CommerceValidationError` carries a structured `issues` array:

```ts
class CommerceValidationError extends CommerceError {
  readonly issues: ValidationIssue[]
}

type ValidationIssue = {
  path: (string | number)[]
  message: string
  code: string
}
```

All error classes are exported from `commerce-kit` (server) and re-exported from `@commerce-kit/client`.

---

## Core error codes

| Code | Class | Status |
|---|---|---|
| `NOT_FOUND` | `CommerceNotFoundError` | 404 |
| `VALIDATION_ERROR` | `CommerceValidationError` | 400 |
| `UNAUTHORIZED` | `CommerceUnauthorizedError` | 401 |
| `FORBIDDEN` | `CommerceForbiddenError` | 403 |
| `INVALID_TRANSITION` | `CommerceStateError` | 422 |
| `INVALID_ORDER` | `CommerceOrderError` | 422 |
| `PAYMENT_FAILED` | `CommercePaymentError` | 402 |
| `PAYMENT_CAPTURE_FAILED` | `CommercePaymentError` | 402 |
| `PAYMENT_REFUND_FAILED` | `CommercePaymentError` | 402 |
| `CURRENCY_MISMATCH` | `MoneyCurrencyError` | 422 |
| `CONFIG_ERROR` | `CommerceConfigError` | 500 |
| `PLUGIN_ERROR` | `CommercePluginError` | 500 |
| `WEBHOOK_SIGNATURE_INVALID` | `CommerceUnauthorizedError` | 401 |

Plugins extend this set using namespaced codes — see [Plugin error codes](#plugin-error-codes).

---

## HTTP error response shape

All errors serialize to a consistent JSON envelope.

```ts
// standard error
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order ord_123 not found",
    "status": 404
  }
}

// validation error (400) — adds issues
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "status": 400,
    "issues": [
      {
        "path": ["items", 0, "quantity"],
        "message": "Must be greater than 0",
        "code": "too_small"
      }
    ]
  }
}
```

---

## Server SDK — throws

Direct server-side calls throw on failure. This is idiomatic for server code that lives inside its own error boundary.

```ts
// throws CommerceNotFoundError if not found
const order = await commerce.orders.get({ id: 'ord_123' })

// throws CommerceStateError on invalid transition
await commerce.orders.cancel({ id: 'ord_123' })
```

Catching on the server:

```ts
import { CommerceError, CommerceStateError } from 'commerce-kit'

try {
  await commerce.orders.cancel({ id: 'ord_123' })
} catch (err) {
  if (err instanceof CommerceStateError) {
    // handle invalid transition
  }
  throw err
}
```

---

## Client SDK — `{ data, error }` result

Client SDK operations never throw. Every operation returns a `CommerceResult<T>`:

```ts
type CommerceResult<T> = {
  data: T | null
  error: CommerceError | null
}
```

`data` is `null` when `error` is set. `error` is `null` when `data` is set. TypeScript narrows both after the error check.

```ts
const { data, error } = await client.orders.checkout({
  items: cart.items,
  fulfillmentMethodId: 'method_shipping',
  payment: { providerId: 'moyasar' },
})

if (error) {
  // error is typed CommerceError — never unknown
  if (error instanceof CommercePaymentError) {
    showPaymentDeclined(error.message)
    return
  }
  if (error instanceof CommerceValidationError) {
    error.issues.forEach(i => showFieldError(i.path, i.message))
    return
  }
  showGenericError(error.code, error.message)
  return
}

// data is fully narrowed here — TypeScript knows error is null
console.log(data.order.id)
```

All client operations follow the same shape:

```ts
const { data, error } = await client.products.list({ where: { status: 'active' } })
const { data, error } = await client.cart.add({ variantId: 'var_123', quantity: 1 })
const { data, error } = await client.orders.calculate({ items: cart.items })
```

### Checking error codes

When `instanceof` is too broad, check `error.code` directly:

```ts
if (error?.code === 'coupons:COUPON_EXPIRED') {
  showCouponExpiredMessage()
}
```

---

## Framework wrappers

`@commerce-kit/react` (and equivalent packages for other frameworks) wrap the base `CommerceResult<T>` to add `isLoading` and reactive state:

```ts
// @commerce-kit/react
const { data, error, isLoading } = useCheckout()

// isLoading is true while the async operation is in flight
// data and error follow the same CommerceResult<T> shape
```

`isLoading` does not exist on the raw `CommerceResult<T>` type because `await` already guarantees the operation is complete. Framework wrappers manage the loading state independently.

---

## Throwing from hooks and plugins

Hooks and plugins throw to abort an operation and return a structured error to the caller.

```ts
import { CommerceError, CommerceForbiddenError } from 'commerce-kit'

// generic with explicit status
throw new CommerceError('Account suspended', {
  code: 'ACCOUNT_SUSPENDED',
  status: 403,
})

// typed class — status inferred
throw new CommerceForbiddenError('Account suspended', {
  code: 'ACCOUNT_SUSPENDED',
})
```

The thrown error is serialized to the HTTP response envelope and surfaced as `error` in the client SDK result. No special handling is required in the calling code beyond checking `error` as usual.

---

## Plugin error codes

Plugin error codes are prefixed with the plugin id to prevent collisions with core codes and with other plugins.

Format: `pluginId:CODE`

```ts
// coupons plugin
throw new CommerceError('Coupon has expired', {
  code: 'coupons:COUPON_EXPIRED',
  status: 422,
})

// inventory plugin
throw new CommerceError('Insufficient stock', {
  code: 'inventory:OUT_OF_STOCK',
  status: 422,
  data: { variantId: 'var_123', requested: 3, available: 1 },
})
```

Plugin-defined codes must be documented in the plugin's own documentation. Core does not maintain a registry of plugin codes.

---

## Cross-links

- HTTP status mapping per error class: [80-framework-adapters.md](./80-framework-adapters.md)
- Throwing errors from before hooks: [42-hooks.md](./42-hooks.md)
- `CommerceResult<T>` in the type inference model: [60-type-inference.md](./60-type-inference.md)
