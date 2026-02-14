---
name: designing-apis
description: Designs REST APIs with Zod validation, consistent error responses, and App Router patterns. Use when creating route handlers, API endpoints, or server-only utilities.
---

# API Design Patterns

REST API design for Next.js App Router with Zod validation and consistent error handling.

**All request/response validation uses Zod schemas.** No other validation libraries.

## Contents

- [Core Principles](#core-principles)
- [File Structure](#file-structure)
- [Route Handler Pattern](#route-handler-pattern)
- [Request Validation](#request-validation)
- [Error Response Format](#error-response-format)
- [HTTP Status Codes](#http-status-codes)
- [Server-Only Code](#server-only-code)
- [Anti-Patterns](#anti-patterns)

---

## Core Principles

1. **Boring and predictable** - Users should know how to use an API before reading docs
2. **Never break userspace** - Once published, breaking changes break users
3. **Full echo pattern** - Return complete resource after creation, including applied defaults
4. **Additive changes only** - Add new fields, never remove or rename existing ones

---

## File Structure

```
src/
├── app/
│   └── api/
│       └── {resource}/
│           ├── route.ts                    # Route handler
│           └── {resource}.types.ts         # Zod schemas + types for this endpoint
│
├── lib/
│   ├── server/                             # Server-only code (import 'server-only')
│   │   ├── auth/                           # Authentication utilities
│   │   └── {domain}/                       # Other server-only domains
│   │
│   ├── services/                           # External API integrations (existing)
│   │   ├── hydra/
│   │   ├── cosmic/
│   │   └── ...
│   │
│   └── api/                                # Shared API utilities
│       ├── api-error.types.ts              # Standard error type
│       └── create-error-response.ts        # Error response helper
```

### Naming Conventions

| Type           | Pattern                               | Example                                   |
| -------------- | ------------------------------------- | ----------------------------------------- |
| Route handler  | `src/app/api/{resource}/route.ts`     | `src/app/api/items/route.ts`              |
| Types file     | `{resource}.types.ts`                 | `items.types.ts`                          |
| Server utility | `src/lib/server/{domain}/{action}.ts` | `src/lib/server/auth/validate-api-key.ts` |
| Request type   | `{Action}{Resource}RequestType`       | `CreateItemRequestType`                   |
| Response type  | `{Action}{Resource}ResponseType`      | `CreateItemResponseType`                  |

---

## Route Handler Pattern

```typescript
// src/app/api/items/route.ts
import { NextRequest } from 'next/server'
import { fetchItems } from '@/lib/server/items/fetch-items'

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const category = searchParams.get('category')

  const items = await fetchItems({ category })

  return Response.json(items, {
    headers: { 'Cache-Control': 's-maxage=3600' },
  })
}
```

---

## Request Validation with Zod

**ALWAYS** use Zod's `safeParse()` immediately after reading request body. Zod is the only validation library used for API request/response validation.

```typescript
import { NextRequest } from 'next/server'
import { CreateItemRequestSchema } from './items.types'
import { createErrorResponse } from '@/lib/api/create-error-response'

export async function POST(request: NextRequest) {
  const body = await request.json().catch(() => ({}))
  const parseResult = CreateItemRequestSchema.safeParse(body)

  if (!parseResult.success) {
    return createErrorResponse({
      code: 'VALIDATION_ERROR',
      message: 'Request validation failed',
      context: parseResult.error.flatten(),
      status: 400,
    })
  }

  // parseResult.data is now validated and typed
  const item = await createItem(parseResult.data)

  return Response.json(item, { status: 201 })
}
```

### Zod Schema File Pattern

Colocate Zod schemas with route handlers in a `.types.ts` file. Derive TypeScript types from schemas using `z.infer`. Type names must end with `Type` suffix.

```typescript
// src/app/api/items/items.types.ts
import { z } from 'zod'

export const CreateItemRequestSchema = z.object({
  name: z.string().min(1),
  description: z.string().optional(),
  quantity: z.number().int().positive(),
  metadata: z.record(z.string()).optional(),
})

export type CreateItemRequestType = z.infer<typeof CreateItemRequestSchema>

export const CreateItemResponseSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  description: z.string().nullable(),
  quantity: z.number(),
  metadata: z.record(z.string()).nullable(),
  createdAt: z.string().datetime(),
})

export type CreateItemResponseType = z.infer<typeof CreateItemResponseSchema>
```

### Query Parameter Validation

Use Zod for query parameters too:

```typescript
import { NextRequest } from 'next/server'
import { z } from 'zod'

const QueryParamsSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  search: z.string().optional(),
})

export async function GET(request: NextRequest) {
  const searchParams = Object.fromEntries(request.nextUrl.searchParams)
  const parseResult = QueryParamsSchema.safeParse(searchParams)

  if (!parseResult.success) {
    return createErrorResponse({
      code: 'VALIDATION_ERROR',
      message: 'Invalid query parameters',
      context: parseResult.error.flatten(),
      status: 400,
    })
  }

  const { page, limit, search } = parseResult.data
  // ...
}
```

---

## Error Response Format

**ALWAYS** return errors with this structure:

```typescript
type ApiError = {
  code: string // Machine-readable error code
  message: string // Human-readable, consistent across similar errors
  detail?: string // Specific problem description and how to fix
  context?: unknown // Structured data (validation errors, etc.)
}
```

### Error Codes

| Code               | When to use                            |
| ------------------ | -------------------------------------- |
| `VALIDATION_ERROR` | Request body/params don't match schema |
| `INVALID_JSON`     | Request body is not valid JSON         |
| `MISSING_PARAMS`   | Required query params missing          |
| `NOT_FOUND`        | Resource doesn't exist                 |
| `UNAUTHORIZED`     | Authentication required                |
| `FORBIDDEN`        | Authenticated but not authorized       |
| `INTERNAL_ERROR`   | Unexpected server error                |

### Error Response Helper

```typescript
// src/lib/api/create-error-response.ts
type CreateErrorResponseParams = {
  code: string
  message: string
  detail?: string
  context?: unknown
  status: number
}

export function createErrorResponse({
  code,
  message,
  detail,
  context,
  status,
}: CreateErrorResponseParams) {
  return Response.json({ code, message, detail, context }, { status })
}
```

---

## HTTP Status Codes

| Code | Use Case                                     |
| ---- | -------------------------------------------- |
| 200  | Successful GET, PUT                          |
| 201  | Successful POST (resource created)           |
| 202  | Accepted for async processing                |
| 400  | Validation error, malformed request          |
| 401  | Authentication required                      |
| 403  | Forbidden (authenticated but not authorized) |
| 404  | Resource not found                           |
| 409  | Conflict (e.g., duplicate idempotency key)   |
| 422  | Unprocessable entity (business logic error)  |
| 429  | Rate limited                                 |
| 500  | Server error                                 |

---

## Server-Only Code

Use `import 'server-only'` to ensure code never gets bundled in the client build.

```typescript
// src/lib/server/auth/validate-api-key.ts
import 'server-only'

type ValidateApiKeyParams = {
  key: string | null
}

export function validateApiKey({ key }: ValidateApiKeyParams) {
  if (!key) return false
  return key === process.env.NEXT_PRIVATE_API_KEY
}
```

### `import 'server-only'` vs `'use server'`

| Pattern                | Purpose                                                                                            |
| ---------------------- | -------------------------------------------------------------------------------------------------- |
| `import 'server-only'` | **Build-time guard** - throws error if imported by client code. Use for utilities with secrets.    |
| `'use server'`         | **Server Actions** - functions callable from client but execute on server. Use for form mutations. |

### When to Use `import 'server-only'`

- Database connections
- API keys and secrets
- Server-side integrations
- Authentication utilities
- Any code that should NEVER reach the client bundle

**Note:** `server-only` throws errors when code is run outside Next.js (e.g., standalone scripts with `tsx`). If server code is also used by scripts, rely on:

- `NEXT_PRIVATE_` env var prefix (Next.js excludes from client)
- `src/lib/server/` folder convention
- Node.js-only dependencies that can't run in browsers

### Environment Variables

Use `NEXT_PRIVATE_` prefix for server-only secrets:

```typescript
// ✅ Good: NEXT_PRIVATE_ ensures Next.js excludes from client
const apiKey = process.env.NEXT_PRIVATE_API_KEY

// ❌ Bad: Could accidentally leak to client
const apiKey = process.env.API_KEY
```

---

## Dynamic Route Parameters

```typescript
// src/app/api/items/[id]/route.ts
import { NextRequest } from 'next/server'
import { createErrorResponse } from '@/lib/api/create-error-response'

type RouteParams = {
  params: Promise<{ id: string }>
}

export async function GET(request: NextRequest, { params }: RouteParams) {
  const { id } = await params

  const item = await getItem({ id })

  if (!item) {
    return createErrorResponse({
      code: 'NOT_FOUND',
      message: 'Item not found',
      status: 404,
    })
  }

  return Response.json(item)
}
```

---

## Caching

```typescript
// Static cache for 12 hours
export async function GET() {
  const data = await fetchData()
  return Response.json(data, {
    headers: { 'Cache-Control': 's-maxage=43200' },
  })
}

// Or use route segment config
export const revalidate = 43200 // 12 hours

export async function GET() {
  const data = await fetchData()
  return Response.json(data)
}
```

---

## Idempotency Keys

For operations that shouldn't be duplicated (orders, payments):

```typescript
export async function POST(request: NextRequest) {
  const body = await request.json().catch(() => ({}))
  const parseResult = CreateOrderRequestSchema.safeParse(body)

  if (!parseResult.success) {
    return createErrorResponse({
      code: 'VALIDATION_ERROR',
      message: 'Request validation failed',
      context: parseResult.error.flatten(),
      status: 400,
    })
  }

  const { idempotencyKey } = parseResult.data

  if (idempotencyKey) {
    const existing = await findOrderByIdempotencyKey({ key: idempotencyKey })
    if (existing) {
      return Response.json(existing, { status: 200 })
    }
  }

  const order = await createOrder(parseResult.data)
  return Response.json(order, { status: 201 })
}
```

---

## Anti-Patterns

| Anti-Pattern                               | Why It's Bad                                  |
| ------------------------------------------ | --------------------------------------------- |
| Using `.parse()` instead of `.safeParse()` | Throws exceptions instead of returning result |
| Different error shapes per endpoint        | Consumers can't handle errors uniformly       |
| Putting secrets in query params            | URLs are logged, cached, visible in history   |
| Auto-incrementing IDs in URLs              | Enables enumeration attacks                   |
| Deeply nested responses (>3 levels)        | Hard to parse, version, extend                |
| Breaking changes without versioning        | Breaks existing consumers silently            |
| Swallowing errors in catch blocks          | Hides failures, prevents debugging            |
| Using `any` for request body               | Defeats type safety, allows invalid data      |

---

## Full Echo Pattern

Return complete resource after creation, including applied defaults:

```typescript
// Request
{ "items": [{ "sku": "PROD-001", "qty": 2 }] }

// Response includes ALL fields + defaults
{
  "id": "order-123",
  "items": [{ "sku": "PROD-001", "qty": 2, "price": 29.99 }],
  "status": "pending",           // Default applied
  "total": 59.98,                // Calculated
  "createdAt": "2024-01-01T12:00:00Z"
}
```

---

## Checklist

- [ ] Route handler in `src/app/api/{resource}/route.ts`
- [ ] Zod schemas colocated in `{resource}.types.ts`
- [ ] Type names end with `Type` suffix (e.g., `CreateItemRequestType`)
- [ ] Request body validated with Zod `safeParse()` immediately after `request.json()`
- [ ] Query params validated with Zod `safeParse()`
- [ ] TypeScript types derived from Zod schemas with `z.infer`
- [ ] Errors use consistent format: `{ code, message, detail?, context? }`
- [ ] Correct HTTP status codes used
- [ ] Server-only code uses `import 'server-only'`
- [ ] Secrets use `NEXT_PRIVATE_` prefix
- [ ] Dynamic params awaited: `const { id } = await params`
- [ ] Full resource returned after creation (full echo pattern)
- [ ] Idempotency keys for non-idempotent operations
- [ ] Cache headers set appropriately
