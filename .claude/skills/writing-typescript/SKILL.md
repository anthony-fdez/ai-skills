---
name: writing-typescript
description: Enforces TypeScript patterns including type inference, union types over overloading, and compile-time safety. Use when writing any TypeScript code across the codebase.
---

# Writing TypeScript

Language-level TypeScript patterns that apply across all code.

## Code Style Rules

```typescript
// ✅ Arrow functions for components
const Component = () => {}

// ✅ Type imports
import type { Props } from './types'

// ✅ Path alias
import { Button } from '@/components/_ui/Button'

// ✅ Single quotes, no semicolons
const name = 'value'
```

## NO COMMENTS Rule

Write self-documenting code. Comments are a code smell indicating unclear naming.

```typescript
// ❌ Bad: Redundant comments
// Get the user name
const userName = user.name

// Loop through items
items.forEach((item) => {
  // Process the item
  processItem(item)
})

// ❌ Bad: JSDoc that just repeats parameter names
/**
 * Checks if an error is a quota error.
 *
 * @param error - The error to check
 * @returns true if the error is a quota error
 */
function isQuotaError(error: unknown): boolean

// ✅ Good: Self-documenting code (no comments needed)
const userName = user.name
items.forEach(processItem)

// ✅ Good: Function name is self-documenting
function isQuotaError(error: unknown): boolean

// ✅ Acceptable: Non-obvious "why"
// Using setTimeout to defer execution until after React's commit phase
setTimeout(() => focusInput(), 0)
```

**Never add JSDoc comments that just describe what the code already shows.** If the function name, parameter names, and types are clear, no comment is needed.

## Prefer Inference Over Explicit Types

```typescript
// ✅ Good: Inferred return type
const add = (a: number, b: number) => {
  return a + b
}

// ❌ Bad: Explicit return type
const add = (a: number, b: number): number => {
  return a + b
}
```

## No Function Overloading

```typescript
// ❌ Bad: Overloading
function mapCustomerType(
  customerType: CustomerType,
  options: { toLowerCase: true },
): string
function mapCustomerType(
  customerType: CustomerType,
  options?: { toLowerCase?: false },
): CustomerType
function mapCustomerType(
  customerType: CustomerType,
  options?: { toLowerCase?: boolean },
) {
  return options?.toLowerCase ? mappedType.toLowerCase() : mappedType
}

// ✅ Good: Split into separate functions
const mapCustomerType = (customerType: CustomerType) => {
  return customerType.toString()
}

const mapCustomerTypeToLowerCase = (customerType: CustomerType) => {
  return mapCustomerType(customerType).toLowerCase()
}
```

## No Type Assertions (as keyword)

```typescript
// ❌ Bad: Type assertion bypasses type checking
return null as unknown as ResponseType  // Extremely dangerous!

// ❌ Bad: Asserting API response type
(responseJson as ApiResponse).data.map(...)

// ✅ Good: Explicit null handling
const sendRequest = async <T>(): Promise<T | null> => {
  if (invalidToken) return null
  return response.json()
}

// ✅ Good: Validate at runtime with Zod
const ApiResponseSchema = z.object({ data: z.array(z.string()) })
const validated = ApiResponseSchema.parse(responseJson)
validated.data.map(...)
```

## No `any` Type

```typescript
// ❌ Bad: any defeats type safety
interface RequestBody {
  properties: { [key: string]: any }
}

// ❌ Bad: any in catch
try {
  await operation()
} catch (error: any) {
  console.error(error.message)
}

// ✅ Good: Specific types or unknown
interface RequestBody {
  properties: Record<string, unknown>
}

// ✅ Good: Type narrowing for errors
try {
  await operation()
} catch (error) {
  if (error instanceof Error) {
    console.error(error.message)
  }
}
```

## No Non-Null Assertions (!)

```typescript
// ❌ Bad: Non-null assertion can crash at runtime
const hydraToken = user.token!
const containerWidth = textRef.current.parentElement!.offsetWidth

// ✅ Good: Explicit null check
if (!user.token) {
  throw new Error('User must be authenticated')
}
const hydraToken = user.token

// ✅ Good: Optional chaining with fallback
const containerWidth = textRef.current?.parentElement?.offsetWidth ?? 0
```

## No Traditional Enums

```typescript
// ❌ Bad: Traditional enum
enum FeaturedProductsTabs {
  BestSellers = 'best-sellers',
  Featured = 'featured',
}

// ✅ Good: Const object with as const
const FeaturedProductsTabs = {
  BestSellers: 'best-sellers',
  Featured: 'featured',
} as const

type FeaturedProductsTab =
  (typeof FeaturedProductsTabs)[keyof typeof FeaturedProductsTabs]

// ✅ Good: Union type (even simpler)
type FeaturedProductsTab = 'best-sellers' | 'featured'
```

## Use Utility Types (Don't Duplicate)

```typescript
// ❌ Bad: Duplicating types
type EcommerceObject = {
  currency: string | undefined
  value: number
  items: Array<...>
}

// ✅ Good: Use ReturnType
const getEcommerceObject = () => ({ currency, value, items })
type EcommerceObject = ReturnType<typeof getEcommerceObject>

// ✅ Good: Use Parameters
const createUser = (name: string, age: number) => {}
type CreateUserParams = Parameters<typeof createUser>

// Other useful utilities: Partial, Pick, Omit, Record, Awaited, NonNullable
```

## Use Discriminated Unions

```typescript
// ❌ Bad: Boolean flags allow impossible states
type LoadingState = {
  isLoading: boolean
  isError: boolean
  data?: string[]
  error?: Error
}
// Can have isLoading=true AND isError=true (impossible!)

// ✅ Good: Discriminated union
type LoadingState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; error: Error }

const handleState = (state: LoadingState) => {
  switch (state.status) {
    case 'idle':
      return 'No data yet'
    case 'loading':
      return 'Loading...'
    case 'success':
      return `Loaded ${state.data.length} items`
    case 'error':
      return `Error: ${state.error.message}`
  }
}
```

## Environment Variables Pattern

```typescript
// ❌ Bad: Defaults hide missing config
const clientToken = process.env.DATADOG_TOKEN || ''
const baseUrl = process.env.HYDRA_URL || 'hydra.default.com'

// ❌ Bad: Environment-based conditional logic
const verbose = process.env.NODE_ENV === 'development'

// ✅ Good: Fail immediately if missing
if (!process.env.DATADOG_TOKEN) {
  throw new Error('DATADOG_TOKEN is not set')
}
const clientToken = process.env.DATADOG_TOKEN

// ✅ Good: Explicit feature flags (not environment names)
if (!process.env.MIXPANEL_VERBOSE_LOGGING) {
  throw new Error('MIXPANEL_VERBOSE_LOGGING is not set')
}
const verbose = process.env.MIXPANEL_VERBOSE_LOGGING === 'true'

// ✅ Good: Flat config matching env var names
const config = {
  DATABASE_URL: process.env.DATABASE_URL,
  ENABLE_SSL: process.env.ENABLE_SSL === 'true',
}
```

## No Magic Strings/Numbers

```typescript
// ❌ Bad: Magic strings
if (urlAlpha3 === 'unsupported' || urlAlpha3 === 'reset')
if (urlAlpha3 === 'c' || urlAlpha3 === 'e')

// ✅ Good: Named constants
const ROUTE_SEGMENTS = {
  COUNTRY_CODE: 'c',
  EVENT: 'e',
  UNSUPPORTED: 'unsupported',
  RESET: 'reset',
} as const

if (urlAlpha3 === ROUTE_SEGMENTS.UNSUPPORTED || urlAlpha3 === ROUTE_SEGMENTS.RESET)
```

## No Nested Ternaries

```typescript
// ❌ Bad: Nested ternaries
const email = isIdRequired ? data.email : isEmail ? data.email : undefined
const id = isIdRequired ? data.id : isEmail ? undefined : data.email

// ✅ Good: Use if-else or extract to function
const getResetPasswordFields = (data, isIdRequired, isEmail) => {
  if (isIdRequired) return { email: data.email, id: data.id }
  if (isEmail) return { email: data.email, id: undefined }
  return { email: undefined, id: data.email }
}

const { email, id } = getResetPasswordFields(data, isIdRequired, isEmail)
```

## Simplify Complex Conditionals

```typescript
// ❌ Bad: Long AND chain
if (
  productCategoryData &&
  !('error' in productCategoryData) &&
  'banners' in productCategoryData &&
  productCategoryData.banners.length > 0
) {
  return productCategoryData.banners[0]
}

// ✅ Good: Early returns with guard clauses
const hasError = !productCategoryData || 'error' in productCategoryData
if (hasError) return null

const hasBanners =
  'banners' in productCategoryData && productCategoryData.banners.length > 0
if (!hasBanners) return null

return productCategoryData.banners[0]
```

## Proper Boolean Handling

```typescript
// ❌ Bad: Truthy/falsy can surprise you
const users: User[] = []
if (users) { /* Always true! Empty arrays are truthy */ }

const count = 0
return <div>{count && <span>{count} items</span>}</div>
// Renders: <div>0</div> (not what you wanted)

// ✅ Good: Explicit boolean checks
if (users.length > 0) { }
return <div>{count > 0 && <span>{count} items</span>}</div>
```

## Use Type Guards

```typescript
// ❌ Bad: Repeated instanceof checks
} catch (error) {
  const errorMessage = error instanceof Error ? error.message : 'Unknown error'
  const isTimeout = error instanceof Error && error.name === 'AbortError'
}

// ✅ Good: Narrow once with type guard
} catch (error) {
  if (!(error instanceof Error)) {
    logError('Non-Error thrown:', error)
    return null
  }

  // Now error is narrowed to Error
  const errorMessage = error.message
  const isTimeout = error.name === 'AbortError'
}
```

## Zod at I/O Boundaries

**Schema is the source of truth for types.** Never use `.pipe(z.custom<HandWrittenType>())` to force a schema's output type to a hand-written interface. Types must be derived via `z.infer<typeof Schema>`. If the app accesses a field, add it to the schema — don't maintain a parallel hand-written type.

```typescript
import { z } from 'zod'

const PaymentSchema = z.object({
  amount: z.number().positive(),
  currency: z.string().length(3),
})

type Payment = z.infer<typeof PaymentSchema>

// Validate at boundary
export const processPayment = (untrusted: unknown) => {
  const payment = PaymentSchema.parse(untrusted) // Throws if invalid
  return handlePayment(payment) // Now trusted
}

// Internal function trusts the types
const handlePayment = (payment: Payment) => {
  return payment.amount * getExchangeRate(payment.currency)
}
```

## Named Object Parameters (ALWAYS)

**CRITICAL:** Always use object destructuring for function parameters. Never use positional arguments.

```typescript
// ❌ Bad: Positional arguments - NEVER do this
function fetchSheetData(sheets, auth, spreadsheetId, sheetName) {}
fetchSheetData(sheets, auth, 'abc123', 'Sheet1')

function isQuotaError(error) {}
isQuotaError(someError)

// ✅ Good: Named object parameters - ALWAYS do this
type FetchSheetDataParams = {
  sheets: SheetsClient
  auth: JWT
  spreadsheetId: string
  sheetName: string
}

function fetchSheetData({
  sheets,
  auth,
  spreadsheetId,
  sheetName,
}: FetchSheetDataParams) {}
fetchSheetData({ sheets, auth, spreadsheetId: 'abc123', sheetName: 'Sheet1' })

function isQuotaError({ error }: { error: unknown }) {}
isQuotaError({ error: someError })
```

**This applies to ALL functions, including:**

- Single parameter functions
- Utility functions
- Helper functions
- Internal functions

**Only exception:** Built-in JavaScript methods (e.g., `Array.map`, `String.includes`)

## Checklist

- [ ] **Named object parameters for ALL functions (not positional)**
- [ ] **No JSDoc comments that repeat what code shows**
- [ ] No explicit return types (let TypeScript infer)
- [ ] No function overloading (use union types)
- [ ] No type assertions (`as`) - use type guards or Zod
- [ ] No `any` type - use `unknown` with narrowing
- [ ] No non-null assertions (`!`) - use explicit checks
- [ ] No traditional enums - use const objects or union types
- [ ] No magic strings/numbers - use named constants
- [ ] No nested ternaries - use if-else or functions
- [ ] Environment variables fail fast (no defaults)
- [ ] Explicit feature flags (not environment name checks)
- [ ] Zod validation at I/O boundaries only
- [ ] Discriminated unions for complex state
- [ ] Utility types used (ReturnType, Parameters, etc.)
