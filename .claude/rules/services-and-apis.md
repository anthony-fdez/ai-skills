---
description: External services, API conventions, and integration paths
globs: ["src/lib/services/**", "src/app/api/**", "src/lib/server/**", "pages/api/**"]
---

# Services & API Conventions

## External Services

| Service    | Purpose                    | Path                          |
| ---------- | -------------------------- | ----------------------------- |
| `hydra`    | Customer, orders, payments | `src/lib/services/hydra/`     |
| `cosmic`   | CMS content, products      | `src/lib/services/cosmic/`    |
| `jeeves`   | Backend services           | `src/lib/services/jeeves/`    |
| `rosetta`  | Translations               | `src/lib/services/rosetta/`   |
| `mixpanel` | Analytics                  | `src/lib/hooks/mixpanel`      |

## API Route Convention

- Pages Router API routes use `.api.ts` extension: `pages/api/endpoint.api.ts`
- App Router routes (if used): `src/app/api/{resource}/route.ts`

## Server-Only Code

- Location: `src/lib/server/{domain}/{action}.ts`
- Use `import 'server-only'` for code with secrets
- Use `NEXT_PRIVATE_` prefix for server-only env vars

## Error Response Helper

Use `createErrorResponse` from `src/lib/api/create-error-response.ts` for consistent API error responses:

```
{ code: string, message: string, detail?: string, context?: unknown }
```

## Service File Naming

Files must include the service prefix: `cosmic-get-all-products.ts`, `hydra-update-customer-cart.ts`
