---
name: writing-logs
description: Enforces structured logging patterns for DataDog including context prefixes in square brackets, tag ordering (group > page > source > operation > status), log levels, and metadata structure. Use when adding any log statements, instrumenting code for observability, debugging with logs, or adding retry logging. Also trigger when reviewing logs for proper tag cardinality, message formatting, error serialization, or when code puts high-cardinality data in tags instead of metadata.
---

# Writing Logs

Patterns for structured logging with DataDog integration.

## Contents

- [Logger API](#logger-api)
- [Message Format](#message-format)
- [Log Params Structure](#log-params-structure)
- [Standardized Datadog Payload](#standardized-datadog-payload)
- [Locale Parameter](#locale-parameter)
- [Variant Parameter](#variant-parameter)
- [Tag Ordering](#tag-ordering)
- [Available Tags](#available-tags)
- [Complete Example](#complete-example)
- [Tag Cardinality Rules](#tag-cardinality-rules)
- [Log Levels](#log-levels)
- [Error Logging](#error-logging)
- [Retry Logging](#retry-logging)
- [Minimal Logging](#minimal-logging-no-context)
- [Adding New Tags](#adding-new-tags)
- [Checklist](#checklist)

---

## Logger API

```typescript
import { logger } from '@/lib/logger'

// Log levels - params object is optional
logger.debug('Verbose debugging info', { metadata: { ... } })
logger.info('Normal operation completed', { metadata: { ... } })
logger.warn('Something unexpected but handled', { metadata: { ... } })
logger.error('Operation failed', { error, metadata: { ... } })

// Minimal log - no params needed
logger.info('Simple log message')

// Access centralized tags
const { PAGE, STATUS, SOURCE, OPERATION, GROUP, RETRY, ACTION } = logger.tags
```

---

## Message Format

All log messages MUST start with a context prefix in square brackets:

```typescript
// Pattern: '[Context] Actual message'

// ✅ Good: Clear context prefix
logger.info('[Products Grid] Build completed', { ... })
logger.error('[Product Page] Data fetch failed', { ... })
logger.warn('[Build: Categories] Falling back to defaults', { ... })
logger.info('[Build: Retry] cms:getProduct succeeded after 2 retries', { ... })

// ❌ Bad: No context prefix
logger.info('Products grid page build completed', { ... })
logger.error('Error fetching CMS modes:', { ... })
```

**Context naming conventions:**

- Use Title Case for the context name
- Use colons for sub-contexts: `[Apple Pay: Session]`, `[Build: Retry]`
- Keep context concise but descriptive
- Match context to the feature/area, not the function name

---

## Log Params Structure

```typescript
type LogParams = {
  locale?: string | null // Locale code (e.g., 'en-US'), defaults to null
  variant?: string | null // Site variant (e.g., 'premium'), null for default
  tags?: LogTag[] // Array of typed tags, defaults to []
  source?: LogSource // 'page-build' | 'cms' | 'backend-api' | 'i18n' | 'payments-api' | 'client'
  operation?: string // Operation name for debugging
  metadata?: Record<string, unknown> // Custom data (durationMs, slugs, etc.)
  error?: unknown // Error object - will be serialized
}
```

All fields are optional. The params object itself is also optional. The logger automatically:

- Defaults `locale` to `null` if not provided
- Defaults `variant` to `null` if not provided
- Prepends `env` and `phase` tags automatically

---

## Standardized Datadog Payload

All logs are sent to Datadog with a **fixed, predictable structure**. This makes it easy to query and build dashboards.

```typescript
// What gets sent to Datadog (context object)
type LogContext = {
  locale: string | null // Always at @context.locale
  variant: string | null // Always at @context.variant
  source: LogSource | null // Always at @context.source
  operation: string | null // Always at @context.operation
  operationStatus: string | null // Extracted from status:* tag
  error: {
    name: string
    message: string
    stack: string | null
    httpStatus: number | null // HTTP status for fetch errors
    rawResponse: unknown // Raw response body for debugging
  } | null
  metadata: Record<string, unknown> | null
  // Plus parsed tags: group, page, status, retry, action, etc.
}
```

**Key benefits:**

- **No duplicate fields** - `source` is always at `@context.source`
- **Error always normalized** - Consistent shape with httpStatus extraction
- **Metadata isolated** - Custom data in `@context.metadata`
- **Tags parsed** - `@context.group`, `@context.status`, etc. for easy queries

```typescript
// ✅ Good: Data in correct locations
logger.error('[Product Page] Fetch failed', {
  locale,
  variant,
  source: 'cms', // → @context.source
  operation: 'getProduct', // → @context.operation
  metadata: { productSlug }, // → @context.metadata.productSlug
  error: fetchError, // → @context.error.{name,message,stack,httpStatus}
})

// ❌ Bad: Don't put source/operation in metadata
logger.error('[Product Page] Fetch failed', {
  metadata: {
    source: 'cms', // Wrong! Use params.source instead
    productSlug,
  },
})
```

---

## Locale Parameter

The `locale` parameter identifies the user's locale (e.g., 'en-US', 'de-DE'):

```typescript
// ✅ Good: Locale as a parameter
logger.info('[Product Page] Product fetched', {
  locale, // Locale code like 'en-US'
  tags: [PAGE.PRODUCT, STATUS.SUCCESS],
})

// ✅ Good: No locale (defaults to null)
logger.info('[App] Initialized')

// ❌ Bad: Don't create locale tags manually
logger.info('[Product Page] Product fetched', {
  tags: ['locale:en-us', PAGE.PRODUCT], // Wrong! Locale goes in params
})
```

---

## Variant Parameter

The `variant` identifies which site variant the log is for:

```typescript
// ✅ Good: Include variant when available
logger.info('[Product Page] Build completed', {
  locale,
  variant: 'premium', // or null for default
  tags: [GROUP.PRODUCT_BUILD, STATUS.SUCCESS],
})

// Useful for filtering logs by variant in DataDog:
// @context.variant:premium
```

---

## Tag Ordering

Tags follow this order for consistent DataDog filtering:

```text
1. env        (automatic - prepended by logger)
2. phase      (automatic - prepended by logger)
3. group      (feature group: product-build, apple-pay, auth, revalidation)
4. page       (which page: product, products-grid, category)
5. source     (which service: cms, backend-api, payments-api, i18n)
6. operation  (what action: fetch-product, fetch-pricing)
7. retry      (retry state: attempt, success, exhausted)
8. action     (special action: client-fallback, error-boundary)
9. status     (outcome: success, error, not-found) - ALWAYS LAST
```

### Automatic Tags

The logger automatically prepends these tags:

- **env** - Based on `NEXT_PUBLIC_APP_ENV` (local, preview, staging, production)
- **phase** - Based on `NEXT_PHASE` (build vs runtime)

### Example with Correct Order

```typescript
const { GROUP, PAGE, SOURCE, OPERATION, STATUS } = logger.tags

// ✅ Good: Tags in correct order, status last
logger.info('[Product Page] Data fetch started', {
  locale,
  variant,
  tags: [
    GROUP.PRODUCT_BUILD,
    PAGE.PRODUCT,
    SOURCE.CMS,
    OPERATION.FETCH_PRODUCT,
  ],
})

logger.error('[Product Page] Fetch failed', {
  locale,
  variant,
  tags: [
    GROUP.PRODUCT_BUILD,
    PAGE.PRODUCT,
    SOURCE.CMS,
    OPERATION.FETCH_PRODUCT,
    STATUS.ERROR, // Status always last
  ],
  error,
})

// ❌ Bad: Tags out of order
logger.error('[Product Page] Fetch failed', {
  tags: [STATUS.ERROR, SOURCE.CMS, PAGE.PRODUCT], // Wrong order!
})
```

---

## Available Tags

Access via `logger.tags`:

```typescript
const {
  ENV, // Auto-added, don't use manually
  PHASE, // Auto-added, don't use manually
  GROUP,
  PAGE,
  SOURCE,
  OPERATION,
  RETRY,
  ACTION,
  STATUS,
} = logger.tags

// Group (feature area)
GROUP.PRODUCT_BUILD // 'group:product-build'
GROUP.EXTERNAL_API // 'group:external-api'
GROUP.APPLE_PAY // 'group:apple-pay'
GROUP.AUTH // 'group:auth'
GROUP.CLIENT_ERROR // 'group:client-error'
GROUP.REVALIDATION // 'group:revalidation'

// Page/Feature
PAGE.PRODUCT // 'page:product'
PAGE.PRODUCTS_GRID // 'page:products-grid'
PAGE.CATEGORY // 'page:category'

// Source Service
SOURCE.PAGE_BUILD // 'source:page-build'
SOURCE.CMS // 'source:cms'
SOURCE.BACKEND // 'source:backend-api'
SOURCE.I18N // 'source:i18n'
SOURCE.PAYMENTS // 'source:payments-api'
SOURCE.CLIENT // 'source:client'

// Operation
OPERATION.FETCH_PRODUCT // 'op:fetch-product'
OPERATION.FETCH_PRICING // 'op:fetch-pricing'
OPERATION.FETCH_TRANSLATIONS // 'op:fetch-translations'
OPERATION.FETCH_SLUGS // 'op:fetch-slugs'
OPERATION.FETCH_CATEGORY // 'op:fetch-category'

// Retry (for retry mechanism logs)
RETRY.ATTEMPT // 'retry:attempt'
RETRY.SUCCESS // 'retry:success'
RETRY.EXHAUSTED // 'retry:exhausted'

// Action (special actions)
ACTION.CLIENT_FALLBACK // 'action:client-fallback'
ACTION.ERROR_BOUNDARY // 'action:error-boundary'

// Status (ALWAYS LAST in tag array)
STATUS.SUCCESS // 'status:success'
STATUS.ERROR // 'status:error'
STATUS.NOT_FOUND // 'status:not-found'
STATUS.UNAVAILABLE // 'status:unavailable'
```

---

## Complete Example

```typescript
import { logger, LOG_SOURCE } from '@/lib/logger'

const { GROUP, PAGE, SOURCE, OPERATION, STATUS } = logger.tags

export const getProductPageData = async ({
  productSlug,
  locale,
  variant,
}) => {
  const context = { productSlug, locale }
  const startTime = Date.now()

  try {
    const product = await fetchProduct(productSlug)

    logger.info('[Product Page] CMS fetch completed', {
      locale,
      variant,
      tags: [
        GROUP.PRODUCT_BUILD,
        PAGE.PRODUCT,
        SOURCE.CMS,
        OPERATION.FETCH_PRODUCT,
        STATUS.SUCCESS,
      ],
      source: LOG_SOURCE.CMS,
      operation: 'getProductPageData',
      metadata: {
        ...context,
        hasVariations: product.variations.length > 0,
        durationMs: Date.now() - startTime,
      },
    })

    return { data: product, error: null }
  } catch (error) {
    logger.error('[Product Page] Failed to fetch from CMS', {
      locale,
      variant,
      tags: [
        GROUP.PRODUCT_BUILD,
        PAGE.PRODUCT,
        SOURCE.CMS,
        OPERATION.FETCH_PRODUCT,
        STATUS.ERROR,
      ],
      source: LOG_SOURCE.CMS,
      operation: 'getProductPageData',
      metadata: {
        ...context,
        durationMs: Date.now() - startTime,
      },
      error,
    })

    return { data: null, error: createFetchError(error) }
  }
}
```

---

## Tag Cardinality Rules

**Low cardinality (use tags):** Fixed set of known values

- Group, page, source, operation, status, retry, action

**High cardinality (use metadata):** Variable/dynamic values

- Product slugs, user IDs, SKUs, timestamps, error messages, durations

```typescript
// ✅ Good: High cardinality in metadata
logger.info('[Product Page] Fetched', {
  locale,
  tags: [PAGE.PRODUCT, STATUS.SUCCESS],
  metadata: { productSlug, sku, userId, durationMs },
})

// ❌ Bad: High cardinality in tags
logger.info('[Product Page] Fetched', {
  tags: [`product:${productSlug}`, `sku:${sku}`], // Creates too many unique tags!
})
```

---

## Log Levels

| Level   | Use For                      | Example                            |
| ------- | ---------------------------- | ---------------------------------- |
| `debug` | Verbose info for debugging   | Variable values, flow tracing      |
| `info`  | Normal operations            | "CMS fetch completed"              |
| `warn`  | Unexpected but handled       | "Product not found, returning 404" |
| `error` | Failures requiring attention | "API call failed after retries"    |

### Don't Over-Log

```typescript
// ❌ Bad: Logging every step
logger.info('[Fetch] Starting fetch')
logger.info('[Fetch] Building URL')
logger.info('[Fetch] Sending request')

// ✅ Good: Log meaningful events
logger.info('[Product Page] CMS fetch completed', {
  locale,
  tags: [PAGE.PRODUCT, SOURCE.CMS, STATUS.SUCCESS],
  metadata: { productSlug, variationCount: skus.length, durationMs },
})
```

---

## Error Logging

Always include the error object - the logger serializes it properly:

```typescript
try {
  await fetchProduct()
} catch (error) {
  // ✅ Good: Include error object
  logger.error('[Product Page] Failed to fetch', {
    locale,
    tags: [PAGE.PRODUCT, SOURCE.CMS, STATUS.ERROR],
    metadata: { productSlug },
    error, // Serialized to: name, message, stack, httpStatus, rawResponse
  })
}
```

The logger extracts `httpStatus` from common error properties (`error.httpStatus`, `error.status`, `error.statusCode`) and `rawResponse` if present.

---

## Retry Logging

Use `createRetryLogger` for operations with retry logic:

```typescript
import { createRetryLogger } from '@/lib/build/create-retry-logger'

const onRetryEvent = createRetryLogger({ locale })

await retryWithBackoff(() => cms.getProductData({ productSlug }), {
  shouldRetry: shouldRetryServerErrors,
  operationName: `cms:getProduct:${productSlug}`,
  onRetryEvent, // Logs retry:attempt, retry:success, retry:exhausted
})
```

The retry logger automatically:

- Skips logging for first-attempt successes (no retries needed)
- Logs `retry:attempt` with attempt number and delay
- Logs `retry:success` when operation succeeds after retries
- Logs `retry:exhausted` when all retries fail

---

## Minimal Logging (No Context)

For simple logs without market/tag context:

```typescript
// ✅ Valid: No params needed
logger.info('[App] Started')
logger.warn('[Config] Feature flag not found')
logger.error('[Unexpected] Unknown error', { error })
```

---

## Adding New Tags

When adding new tags to `logger-tags.ts`:

1. Add to the appropriate category constant (GROUP, PAGE, SOURCE, OPERATION, etc.)
2. Use lowercase with hyphens for values: `'group:new-feature'`
3. The `LogTag` type union updates automatically via `ValueOf<typeof CATEGORY>`
4. Update this skill file with the new tag

```typescript
// In logger-tags.ts
export const GROUP = {
  PRODUCT_BUILD: 'group:product-build',
  NEW_FEATURE: 'group:new-feature', // New tag
} as const
```

---

## Checklist

- [ ] Log message starts with `[Context]` prefix in Title Case
- [ ] `locale` parameter used when locale context is available
- [ ] `variant` parameter used when variant context is available
- [ ] Tags follow correct order: group → page → source → operation → retry/action → status
- [ ] Status tag is LAST in the tags array
- [ ] Using `logger.tags.*` constants (no hardcoded tag strings)
- [ ] NOT manually adding env or phase tags (they're automatic)
- [ ] High cardinality data in `metadata`, not `tags`
- [ ] `durationMs` included in metadata for timed operations
- [ ] Error object passed to `error` param (not stringified)
- [ ] `source` param matches the SOURCE tag used
- [ ] `operation` param describes the function/action
- [ ] Log level matches severity (debug/info/warn/error)
- [ ] Meaningful log messages (not just "Error" or "Success")
- [ ] Not over-logging (one log per meaningful event)
