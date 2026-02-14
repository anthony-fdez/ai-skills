---
name: writing-logs
description: Enforces logging patterns for DataDog including tag ordering, structured metadata, and log levels. Use when adding logs, debugging, or instrumenting code for observability.
---

# Writing Logs

Patterns for structured logging with DataDog integration.

## Contents

- [Logger API](#logger-api)
- [Message Format](#message-format)
- [Log Params Structure](#log-params-structure)
- [Standardized Datadog Payload](#standardized-datadog-payload)
- [Market Parameter](#market-parameter)
- [Mode Slug Parameter](#mode-slug-parameter)
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
logger.info('[Build: Retry] cosmic:getProduct succeeded after 2 retries', { ... })

// ❌ Bad: No context prefix
logger.info('Products grid page build completed', { ... })
logger.error('Error fetching cosmic modes:', { ... })
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
  market?: SupportedAlpha2CountryCode | null // Alpha2 country code (e.g., 'US'), defaults to null
  modeSlug?: string | null // Mode slug (e.g., 'ufeelgreat'), null for default shop
  tags?: LogTag[] // Array of typed tags, defaults to []
  source?: LogSource // 'page-build' | 'cosmic' | 'jeeves' | 'rosetta' | 'hydra' | 'client'
  operation?: string // Operation name for debugging
  metadata?: Record<string, unknown> // Custom data (durationMs, slugs, etc.)
  error?: unknown // Error object - will be serialized
}
```

All fields are optional. The params object itself is also optional. The logger automatically:

- Defaults `market` to `null` if not provided
- Defaults `modeSlug` to `null` if not provided
- Prepends `env` and `phase` tags automatically

---

## Standardized Datadog Payload

All logs are sent to Datadog with a **fixed, predictable structure**. This makes it easy to query and build dashboards.

```typescript
// What gets sent to Datadog (context object)
type LogContext = {
  market: string | null // Always at @context.market
  modeSlug: string | null // Always at @context.modeSlug
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
  market: alpha2,
  modeSlug,
  source: 'cosmic', // → @context.source
  operation: 'getProduct', // → @context.operation
  metadata: { productSlug }, // → @context.metadata.productSlug
  error: fetchError, // → @context.error.{name,message,stack,httpStatus}
})

// ❌ Bad: Don't put source/operation in metadata
logger.error('[Product Page] Fetch failed', {
  metadata: {
    source: 'cosmic', // Wrong! Use params.source instead
    productSlug,
  },
})
```

---

## Market Parameter

The `market` parameter uses Alpha2 country codes (e.g., 'US', 'GB', 'DE'):

```typescript
// ✅ Good: Market as a parameter
logger.info('[Product Page] Product fetched', {
  market: alpha2, // Alpha2 code like 'US'
  tags: [PAGE.PRODUCT, STATUS.SUCCESS],
})

// ✅ Good: No market (defaults to null)
logger.info('[App] Initialized')

// ❌ Bad: Don't create market tags manually
logger.info('[Product Page] Product fetched', {
  tags: ['market:us', PAGE.PRODUCT], // Wrong! Market goes in params
})
```

---

## Mode Slug Parameter

The `modeSlug` identifies which mode/site variant the log is for:

```typescript
// ✅ Good: Include modeSlug when available
logger.info('[Product Page] Build completed', {
  market: alpha2,
  modeSlug: 'ufeelgreat', // or null for default shop
  tags: [GROUP.PRODUCT_BUILD, STATUS.SUCCESS],
})

// Useful for filtering logs by mode in DataDog:
// @context.modeSlug:ufeelgreat
```

---

## Tag Ordering

Tags follow this order for consistent DataDog filtering:

```text
1. env        (automatic - prepended by logger)
2. phase      (automatic - prepended by logger)
3. group      (feature group: product-build, apple-pay, auth, revalidation)
4. page       (which page: product, products-grid, category)
5. source     (which service: cosmic, jeeves, hydra, rosetta)
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
  market: alpha2,
  modeSlug,
  tags: [
    GROUP.PRODUCT_BUILD,
    PAGE.PRODUCT,
    SOURCE.COSMIC,
    OPERATION.FETCH_PRODUCT,
  ],
})

logger.error('[Product Page] Fetch failed', {
  market: alpha2,
  modeSlug,
  tags: [
    GROUP.PRODUCT_BUILD,
    PAGE.PRODUCT,
    SOURCE.COSMIC,
    OPERATION.FETCH_PRODUCT,
    STATUS.ERROR, // Status always last
  ],
  error,
})

// ❌ Bad: Tags out of order
logger.error('[Product Page] Fetch failed', {
  tags: [STATUS.ERROR, SOURCE.COSMIC, PAGE.PRODUCT], // Wrong order!
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
GROUP.HYDRA_API // 'group:hydra-api'
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
SOURCE.COSMIC // 'source:cosmic'
SOURCE.JEEVES // 'source:jeeves'
SOURCE.ROSETTA // 'source:rosetta'
SOURCE.HYDRA // 'source:hydra'
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
  alpha2,
  modeSlug,
}) => {
  const context = { productSlug, locale }
  const startTime = Date.now()

  try {
    const product = await fetchProduct(productSlug)

    logger.info('[Product Page] Cosmic fetch completed', {
      market: alpha2,
      modeSlug,
      tags: [
        GROUP.PRODUCT_BUILD,
        PAGE.PRODUCT,
        SOURCE.COSMIC,
        OPERATION.FETCH_PRODUCT,
        STATUS.SUCCESS,
      ],
      source: LOG_SOURCE.COSMIC,
      operation: 'getProductPageData',
      metadata: {
        ...context,
        hasVariations: product.variations.length > 0,
        durationMs: Date.now() - startTime,
      },
    })

    return { data: product, error: null }
  } catch (error) {
    logger.error('[Product Page] Failed to fetch from Cosmic', {
      market: alpha2,
      modeSlug,
      tags: [
        GROUP.PRODUCT_BUILD,
        PAGE.PRODUCT,
        SOURCE.COSMIC,
        OPERATION.FETCH_PRODUCT,
        STATUS.ERROR,
      ],
      source: LOG_SOURCE.COSMIC,
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
  market: alpha2,
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
| `info`  | Normal operations            | "Cosmic fetch completed"           |
| `warn`  | Unexpected but handled       | "Product not found, returning 404" |
| `error` | Failures requiring attention | "API call failed after retries"    |

### Don't Over-Log

```typescript
// ❌ Bad: Logging every step
logger.info('[Fetch] Starting fetch')
logger.info('[Fetch] Building URL')
logger.info('[Fetch] Sending request')

// ✅ Good: Log meaningful events
logger.info('[Product Page] Cosmic fetch completed', {
  market: alpha2,
  tags: [PAGE.PRODUCT, SOURCE.COSMIC, STATUS.SUCCESS],
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
    market: alpha2,
    tags: [PAGE.PRODUCT, SOURCE.COSMIC, STATUS.ERROR],
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

const onRetryEvent = createRetryLogger({ alpha2 })

await retryWithBackoff(() => cosmic.getProductData({ productSlug }), {
  shouldRetry: shouldRetryServerErrors,
  operationName: `cosmic:getProduct:${productSlug}`,
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
- [ ] `market` parameter used (Alpha2 code) when country context is available
- [ ] `modeSlug` parameter used when mode context is available
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
