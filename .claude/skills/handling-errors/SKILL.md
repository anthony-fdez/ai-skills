---
name: handling-errors
description: Enforces error handling patterns including error bubbling, progressive degradation, and user-friendly messages. Use when handling errors, displaying error states, or implementing error boundaries.
---

# Handling Errors

Patterns for proper error handling including bubbling, progressive degradation, and user-friendly messages.

## Core Principle: Errors Should Bubble Up

```typescript
// ✅ Good: Let errors propagate naturally
const fetchUser = async (id: string) => {
  const response = await fetch(`/api/users/${id}`)

  if (!response.ok) {
    throw new Error(`Failed to fetch user: ${response.statusText}`)
  }

  return response.json()
}

// ❌ Bad: Swallowing errors
const fetchUser = async (id: string) => {
  try {
    const response = await fetch(`/api/users/${id}`)
    return response.json()
  } catch (error) {
    console.log(error) // Swallowed! Caller doesn't know it failed
  }
}
```

## Don't Swallow Errors

### Empty Catch Blocks

```typescript
// ❌ Bad: Empty catch swallows the error
;(async () => {
  await getSubscriptions()
})().catch(() => {}) // Silent failure!

// ✅ Good: Handle or rethrow
try {
  await getSubscriptions()
} catch (error) {
  logError('Failed to fetch subscriptions:', error)
  showErrorToUser('Unable to load subscriptions. Please try again.')
  throw error // Rethrow if caller needs to know
}
```

### Non-Critical Operations

```typescript
// ✅ OK: Logging for non-critical operations (don't rethrow)
try {
  await trackAnalyticsEvent()
} catch (error) {
  logError('Analytics tracking failed (non-critical):', error)
  // Don't throw - analytics failure shouldn't break the app
}
```

## Don't Catch, Mutate, and Return

```typescript
// ❌ Bad: Catching and returning null
const getTopUpOptions = async () => {
  try {
    const response = await fetch(...)
    return await response.json()
  } catch (error) {
    logError('Fetch error:', error)
    return null // Caller can't distinguish "no options" from "error"
  }
}

// ✅ Good: Let error throw or return discriminated union
const getTopUpOptions = async () => {
  const response = await fetch(...)

  if (!response.ok) {
    throw new Error(`Failed to fetch options: ${response.statusText}`)
  }

  return response.json()
}
```

## Async Functions Must Throw Errors

Async functions that fetch data should **throw errors, not return them**. This enables:

1. **Retry mechanisms** to catch and retry on transient failures
2. **Error boundaries** to catch and display errors
3. **Proper error propagation** through the call stack

### Why Throwing Matters

```typescript
// ❌ Bad: Returns error object - retry won't work
const fetchProducts = async () => {
  const response = await fetch('/api/products')

  if (!response.ok) {
    return { error: 'Failed to fetch', httpStatus: response.status }
  }

  return response.json()
}

// Used with retry:
const result = await retryWithBackoff(() => fetchProducts())
// If API returns 500, fetchProducts returns { error: ... }
// retryWithBackoff sees success (no throw), doesn't retry!

// ✅ Good: Throws error - retry works correctly
const fetchProducts = async () => {
  const response = await fetch('/api/products')

  if (!response.ok) {
    throw new Error(`Failed to fetch products: ${response.status}`)
  }

  return response.json()
}

// Used with retry:
const result = await retryWithBackoff(() => fetchProducts())
// If API returns 500, fetchProducts throws
// retryWithBackoff catches, waits, retries ✓
```

### Don't Wrap Functions That Swallow Errors

```typescript
// ❌ Bad: Wrapper swallows errors
const wrapWithErrorHandling =
  (fn) =>
  async (...args) => {
    try {
      return await fn(...args)
    } catch (error) {
      logError(error)
      return null // Swallowed!
    }
  }

const safeFetch = wrapWithErrorHandling(fetchProducts)
// Caller gets null, can't retry, can't distinguish "not found" from "error"

// ✅ Good: Let errors propagate, handle at boundaries
const fetchProducts = async () => {
  const response = await fetch('/api/products')

  if (!response.ok) {
    throw new Error(`Failed: ${response.status}`)
  }

  return response.json()
}

// Handle at the boundary (component, page, service layer)
try {
  const products = await retryWithBackoff(() => fetchProducts())
  return { data: products, error: null }
} catch (error) {
  return { data: null, error: createFetchError(error) }
}
```

### Pattern: Fetch Functions Throw, Services Catch

```
┌─────────────────────────────────────────────────────────────┐
│                    Service Layer                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ try {                                                │    │
│  │   data = await retryWithBackoff(() => fetchFn())     │    │
│  │   return { data, error: null }                       │    │
│  │ } catch (error) {                                    │    │
│  │   return { data: null, error: createError(error) }   │    │
│  │ }                                                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Fetch Function (throws on error)                     │    │
│  │                                                      │    │
│  │ if (!response.ok) throw new Error(...)               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

See: [SSG Error Recovery](../../../docs/features/ssg-error-recovery/ssg-error-recovery.md) for the complete error handling architecture.

## Consistent Error Format

```typescript
// Define error types
type ServiceError = {
  error: true
  message: string
  code?: string
  details?: unknown
}

type ServiceSuccess<T> = {
  error: false
  data: T
}

type ServiceResponse<T> = ServiceError | ServiceSuccess<T>

// Usage
const createOrder = async (order: Order): Promise<ServiceResponse<Order>> => {
  try {
    const result = await api.createOrder(order)
    return { error: false, data: result }
  } catch (e) {
    return {
      error: true,
      message: 'Failed to create order',
      code: 'ORDER_CREATE_FAILED',
    }
  }
}

// Caller handles both cases
const result = await createOrder(order)
if (result.error) {
  showError(result.message)
} else {
  navigateTo(`/orders/${result.data.id}`)
}
```

## Progressive Degradation (Don't Unmount on Error)

### Bad: Unmounting Entire UI

```tsx
// ❌ Bad: Entire checkout disappears on one API failure
const Checkout = () => {
  const { isError } = useGetShippingMethods()

  if (isError) return <ErrorPage /> // Everything gone!

  return (
    <>
      <AccountSection /> {/* User's account info vanishes */}
      <PaymentSection /> {/* Payment methods disappear */}
      <OrderSummary /> {/* Cart contents hidden */}
    </>
  )
}
```

### Good: Show Error Inline, Keep Context

```tsx
// ✅ Good: Keep UI context, show error inline
const CheckoutShipping = () => {
  const {
    data: methods,
    isError,
    error,
    isLoading,
    refetch,
  } = useGetShippingMethods()

  return (
    <section>
      <h2>Shipping</h2>

      {/* Show error but keep UI context */}
      {isError && (
        <ErrorAlert
          title="Unable to load shipping methods"
          message={error?.message}
          action={<Button onClick={() => refetch()}>Retry</Button>}
        />
      )}

      {/* Show loading state */}
      {isLoading && <ShippingMethodsSkeleton />}

      {/* Show data when available */}
      {methods && <ShippingMethodsList methods={methods} />}

      {/* Fallback option */}
      {isError && (
        <Collapsible>
          <ManualShippingEntry />
        </Collapsible>
      )}
    </section>
  )
}
```

### Granular Error Boundaries

```tsx
// ✅ Good: Each section fails independently
const CheckoutPage = () => (
  <Layout>
    <ErrorBoundary fallback={<AccountErrorFallback />}>
      <AccountSection />
    </ErrorBoundary>

    <ErrorBoundary fallback={<ShippingErrorFallback />}>
      <ShippingSection />
    </ErrorBoundary>

    <ErrorBoundary fallback={<PaymentErrorFallback />}>
      <PaymentSection />
    </ErrorBoundary>

    {/* Order summary always visible */}
    <OrderSummary />
  </Layout>
)
```

## User-Friendly Error Messages

### Bad: Vague or Technical Messages

```typescript
// ❌ Bad: Unhelpful error messages
const BadErrorExamples = {
  generic: 'Error: 500 Internal Server Error', // Too technical
  vague: 'Something went wrong', // No actionable info
  blame: 'Invalid input', // Blames user
  deadEnd: 'Failed to load data', // No recovery path
}
```

### Good: Specific and Actionable

```typescript
// ✅ Good: Specific and actionable
const GoodErrorExamples = {
  shipping:
    'Unable to calculate shipping cost. Please verify your address is complete.',
  payment:
    'Payment declined. Please check your card details or try a different payment method.',
  connection:
    'Connection timeout. Please check your internet connection and try again.',
  form: "We couldn't save your changes. Your work is safe - click 'Retry' to save.",
}
```

## ErrorAlert Component Usage

```tsx
import { ErrorAlert } from '@/components/_ui/ErrorAlert'

// Simple error
<ErrorAlert error={{ message: 'Failed to load products' }} />

// With retry action
<ErrorAlert
  error={{
    error: true,
    message: 'Unable to load shipping methods',
    code: 'SHIPPING_FETCH_FAILED',
  }}
  action={
    <Button onClick={refetch} variant="outline" size="sm">
      Retry
    </Button>
  }
/>
```

## Form Error Handling

### Field-Level Errors

```tsx
{
  errors.email && (
    <p className="text-sm text-destructive">
      {translate(errors.email.message)}
    </p>
  )
}
```

### Form-Level Errors (API Errors)

```tsx
const onSubmit = async (data: FormData) => {
  try {
    const response = await submitForm(data)

    if (response.error) {
      setError('root', { message: response.message })
      return
    }

    router.push('/success')
  } catch (error) {
    setError('root', { message: 'Something went wrong. Please try again.' })
  }
}

// Display
{
  errors.root && (
    <Alert variant="destructive">
      <AlertDescription>{errors.root.message}</AlertDescription>
    </Alert>
  )
}
```

## Store Error Handling

```typescript
// Store error in slice with message and code
setCheckout({
  error: {
    message: 'Payment failed',
    code: 'PAYMENT_ERROR',
  },
})

// Display in component
const { error } = useGlobalStore((state) => state.checkout)

{error && (
  <ErrorAlert error={error} />
)}

// Clear error when resolved
setCheckout({ error: null })
```

## Error Type Guards

```typescript
// Type guard for errors with message
const isErrorWithMessage = (error: unknown): error is { message: string } => {
  return (
    typeof error === 'object' &&
    error !== null &&
    'message' in error &&
    typeof error.message === 'string'
  )
}

// Usage
try {
  await someOperation()
} catch (error) {
  if (isErrorWithMessage(error)) {
    showError(error.message)
  } else {
    showError('An unexpected error occurred')
  }
}
```

## Checklist

- [ ] Errors bubble up by default (no empty catch blocks)
- [ ] Async/fetch functions throw errors (don't return error objects)
- [ ] No wrappers that swallow errors and return null
- [ ] Retry mechanisms can catch thrown errors
- [ ] Non-critical operations log errors but don't break the app
- [ ] Error responses use consistent format (ServiceResponse)
- [ ] UI shows errors inline (doesn't unmount entire sections)
- [ ] Error boundaries wrap independent sections
- [ ] Error messages are specific and actionable
- [ ] Form errors set via setError('root', { message })
- [ ] Retry actions provided where appropriate
- [ ] ErrorAlert component used for displaying errors
- [ ] Type guards used for error narrowing
