---
name: handling-errors
description: Enforces error handling patterns including error bubbling, progressive degradation, and user-friendly messages. Use when handling errors, catching exceptions, displaying error states, implementing error boundaries, showing error alerts, writing try/catch blocks, or deciding whether to throw vs return errors. Also trigger when building fallback UI, retry mechanisms, or any time a component needs to gracefully handle a failed API call without unmounting the entire page.
---

# Handling Errors

## Contents

- [Throw Errors, Don't Return Them](#core-principle-errors-bubble-up-throw-dont-return)
- [Use Consistent Error Format](#consistent-error-format)
- [Degrade Progressively (Don't Unmount on Error)](#progressive-degradation-dont-unmount-on-error)
- [Write User-Friendly Error Messages](#user-friendly-error-messages)
- [Use ErrorAlert Component](#erroralert-component-usage)
- [Handle Form Errors with setError](#form-error-handling)
- [Use Error Type Guards](#error-type-guards)

---

## Core Principle: Errors Bubble Up, Throw Don't Return

Async functions that fetch data should **throw errors, not return them**. This enables retry mechanisms, error boundaries, and proper propagation.

```typescript
// ❌ Bad: Returns error — retry won't work, caller can't distinguish failure
const fetchProducts = async () => {
  try {
    const response = await fetch('/api/products')
    return response.json()
  } catch (error) {
    console.log(error)
    return null // Swallowed!
  }
}

// ✅ Good: Throws — retry catches it, error boundaries work
const fetchProducts = async () => {
  const response = await fetch('/api/products')

  if (!response.ok) {
    throw new Error(`Failed to fetch products: ${response.status}`)
  }

  return response.json()
}
```

Non-critical operations (analytics, logging) can catch and log without rethrowing.

### Pattern: Fetch Functions Throw, Services Catch

```typescript
// Service boundary catches and wraps
try {
  const data = await retryWithBackoff(() => fetchProducts())
  return { data, error: null }
} catch (error) {
  return { data: null, error: createFetchError(error) }
}
```

## Consistent Error Format

```typescript
type ServiceError = {
  error: true
  message: string
  code?: string
  details?: unknown
}
type ServiceSuccess<T> = { error: false; data: T }
type ServiceResponse<T> = ServiceError | ServiceSuccess<T>
```

Why: A consistent error shape means every consumer can handle errors the same way — no special-casing per endpoint or service.

## Progressive Degradation (Don't Unmount on Error)

```tsx
// ❌ Bad: Entire page disappears on one API failure
if (isError) return <ErrorPage />

// ✅ Good: Show error inline, keep UI context
<section>
  <h2>Shipping</h2>
  {isError && (
    <ErrorAlert
      title="Unable to load shipping methods"
      message={error?.message}
      action={<Button onClick={() => refetch()}>Retry</Button>}
    />
  )}
  {isLoading && <ShippingMethodsSkeleton />}
  {methods && <ShippingMethodsList methods={methods} />}
</section>

// ✅ Good: Each section fails independently with error boundaries
<ErrorBoundary fallback={<ShippingErrorFallback />}>
  <ShippingSection />
</ErrorBoundary>
```

## User-Friendly Error Messages

```typescript
// ❌ Bad: Technical, vague, or dead-end messages
'Error: 500 Internal Server Error'
'Something went wrong'

// ✅ Good: Specific and actionable
'Unable to calculate shipping cost. Please verify your address is complete.'
'Payment declined. Please check your card details or try a different payment method.'
```

Why: Technical error messages confuse users and leak implementation details. Actionable messages tell users what to do next, reducing support tickets.

## ErrorAlert Component Usage

```tsx
import { ErrorAlert } from '@/components/_ui/ErrorAlert'
;<ErrorAlert
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

```tsx
// API errors → setError('root', { message })
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

// Display with: {errors.root && <Alert variant="destructive">...}
```

## Store Error Handling

```typescript
setState({ error: { message: 'Operation failed', code: 'OPERATION_ERROR' } })
// Display: {error && <ErrorAlert error={error} />}
// Clear: setState({ error: null })
```

## Error Type Guards

```typescript
const isErrorWithMessage = (error: unknown): error is { message: string } => {
  return (
    typeof error === 'object' &&
    error !== null &&
    'message' in error &&
    typeof error.message === 'string'
  )
}
```

## Checklist

- [ ] Async/fetch functions throw errors (don't return error objects)
- [ ] No empty catch blocks or wrappers that swallow errors
- [ ] UI shows errors inline (doesn't unmount entire sections)
- [ ] Error boundaries wrap independent sections
- [ ] Error messages are specific and actionable
- [ ] Retry actions provided where appropriate
- [ ] ErrorAlert component used for displaying errors
