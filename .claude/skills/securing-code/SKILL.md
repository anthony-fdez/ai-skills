---
name: securing-code
description: Implements security patterns including PCI compliance, input validation, and error information leakage prevention. Use when handling sensitive data, payment information, authentication tokens, or writing API endpoints.
---

# Security Patterns

Security patterns for handling sensitive data, preventing information leakage, and protecting API boundaries.

## Contents

- [Redact Sensitive Data from Logs](#redact-sensitive-data-from-logs)
- [Validate Input with Zod at API Boundaries](#validate-input-with-zod-at-api-boundaries)
- [Prevent Error Information Leakage](#prevent-error-information-leakage)
- [Never Expose Tokens in URLs or Client Code](#never-expose-tokens-in-urls-or-client-code)
- [Verify Webhook Signatures Before Processing](#verify-webhook-signatures-before-processing)
- [Handle Payment Security Edge Cases](#handle-payment-security-edge-cases)

---

## Redact Sensitive Data from Logs

**Never log any of the following:**

- Credit card numbers, CVV codes, full account numbers
- Authentication tokens (`token`, `customerToken`)
- Session secrets
- Passwords or security answers
- IP addresses (for privacy)

When logging API responses or request payloads, strip sensitive fields before passing to `logger`:

```typescript
import { logger } from '@/lib/logger'

// GOOD: Log only safe metadata
logger.info('[Checkout] Payment processed', {
  tags: { source: 'hydra', operation: 'create-order' },
  metadata: { orderId: order.id, amount: order.total },
})

// BAD: Logging full API response that may contain card data
logger.info('[Checkout] Payment response', { response: fullApiResponse })
```

Why: PCI DSS requires that cardholder data never appears in logs. DataDog indexes log content, making leaked data searchable and persistent.

---

## Validate Input with Zod at API Boundaries

Use Zod validation at every boundary where untrusted data enters the system: API routes, webhook handlers, and external API responses.

```typescript
// API route — validate incoming request body
const CreateOrderSchema = z.object({
  items: z.array(z.object({ sku: z.string(), quantity: z.number().positive() })),
  shippingAddressId: z.string(),
})

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const parsed = CreateOrderSchema.safeParse(req.body)
  if (!parsed.success) {
    return res.status(400).json(createErrorResponse({
      code: 'VALIDATION_ERROR',
      message: 'Invalid request body',
    }))
  }
  // Use parsed.data — fully typed and validated
}
```

**Validate at boundaries, trust internally:**

- API route handlers — validate `req.body`, `req.query`
- Webhook receivers — validate payload after signature check
- External API responses — validate response shape before using
- **Don't** re-validate data passed between internal functions

Why: Zod at I/O boundaries catches malformed data before it enters the system. Internal code can trust validated types.

---

## Prevent Error Information Leakage

**Never expose to users:**

- Stack traces
- Database queries or connection strings
- Internal API error codes from services (Hydra, Jeeves)
- File paths or server configuration
- Service versions

Use `createErrorResponse` from `src/lib/api/create-error-response.ts` for consistent, safe error responses:

```typescript
// GOOD: Safe error response
return res.status(400).json(createErrorResponse({
  code: 'CARD_DECLINED',
  message: 'Your card was declined. Please try a different payment method.',
}))

// BAD: Leaking internals
return res.status(500).json({
  error: `Hydra API error: ${error.message}`,
  stack: error.stack,
})
```

**Log the full error server-side, return a safe message client-side:**

```typescript
logger.error('[Checkout] Order creation failed', {
  tags: { source: 'hydra', operation: 'create-order', status: 'error' },
  metadata: { orderId, durationMs },
  error,
})

return res.status(500).json(createErrorResponse({
  code: 'ORDER_FAILED',
  message: 'Unable to process your order. Please try again.',
}))
```

Why: Internal details help attackers map your system. User-facing error codes are sufficient for support tickets.

---

## Never Expose Tokens in URLs or Client Code

```typescript
// BAD: Token in URL query parameter
router.push(`/api/order?token=${customerToken}`)

// GOOD: Token in headers
fetch('/api/order', {
  headers: { Authorization: `Bearer ${customerToken}` },
})
```

**Token handling rules:**

- Store tokens in Zustand `user` slice (memory only, not persisted)
- Send tokens via `Authorization` header, never in URL params
- Never log tokens (see redaction rules above)
- Server-only secrets use `NEXT_PRIVATE_` env var prefix and `import 'server-only'`

Why: URL parameters appear in browser history, server logs, and referrer headers. Header-based tokens avoid these leakage vectors.

---

## Verify Webhook Signatures Before Processing

When receiving webhooks from external services, always verify the signature before processing the payload:

```typescript
import { createHmac } from 'crypto'

function verifyWebhookSignature({
  payload,
  signature,
  secret,
  timestamp,
}: {
  payload: string
  signature: string
  secret: string
  timestamp: string
}): boolean {
  const expected = createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex')
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected))
}
```

**Always:**

- Verify signature before any processing
- Use `timingSafeEqual` to prevent timing attacks
- Check timestamp to prevent replay attacks (reject if > 5 minutes old)
- Return 200 quickly, process asynchronously if needed

Why: Without signature verification, anyone can send fake webhook payloads to your endpoint.

---

## Handle Payment Security Edge Cases

### Idempotency

Use idempotency keys for payment operations to prevent duplicate charges:

- Generate a unique key per user action (not per retry)
- Store and reuse the key across retries of the same operation
- Let the payment service handle deduplication

### Race Conditions

- Disable submit buttons while `mutation.isPending` (derive from React Query, don't use separate `useState`)
- Use optimistic locking with version checks for concurrent modifications
- Database-level constraints as the final safety net

### Payment Failures

- Retry with exponential backoff for transient errors (network, timeouts)
- Do NOT retry for validation errors (declined card, insufficient funds)
- Clear, user-friendly error messages without exposing internal details

Why: Payment edge cases can result in double charges, lost transactions, or security vulnerabilities if not handled explicitly.
