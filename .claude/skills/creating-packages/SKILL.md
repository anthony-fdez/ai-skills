---
name: creating-packages
description: Patterns for creating reusable packages with consistent structure, exports, and type conventions. Use when creating a shared package, extracting reusable code, or building API client wrappers.
---

# Creating Packages

Patterns for creating reusable, well-structured packages — whether in a monorepo `packages/` directory or as standalone shared modules.

## Contents

- [Follow Standard Package Structure](#follow-standard-package-structure)
- [Point Types to Source Files for DX](#point-types-to-source-files-for-dx)
- [Choose Entry Point Pattern by Complexity](#choose-entry-point-pattern-by-complexity)
- [Use Workspace References for Internal Dependencies](#use-workspace-references-for-internal-dependencies)
- [Build Client Packages as Thin Shims](#build-client-packages-as-thin-shims)
- [Export Executable Endpoint Functions](#export-executable-endpoint-functions)
- [Name Types by Function Coupling](#name-types-by-function-coupling)
- [Colocate Types with Function Files](#colocate-types-with-function-files)
- [Avoid Filtering Utilities in Client Packages](#avoid-filtering-utilities-in-client-packages)
- [Export Only What the API Provides](#export-only-what-the-api-provides)

---

## Follow Standard Package Structure

```
packages/my-package/
├── src/
│   ├── index.ts          # Main exports
│   ├── types.ts          # Type definitions (if separate export needed)
│   └── {feature}.ts      # One function per file
├── package.json
└── tsconfig.json
```

- One function per file, matching our file naming convention (`{action}-{resource}.ts`)
- `src/index.ts` re-exports the public API
- Keep `README.md` only for packages with complex usage

Why: Consistent structure makes packages predictable and easy to navigate across the codebase.

---

## Point Types to Source Files for DX

In `package.json`, point `types` to source `.ts` files instead of compiled `.d.ts`:

```json
{
  "name": "@scope/my-package",
  "private": true,
  "main": "./dist/index.js",
  "types": "./src/index.ts",
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    }
  }
}
```

Why: IDEs get full type information directly from source. No rebuild needed for type changes to propagate. Go-to-definition jumps to source code, not compiled output.

---

## Choose Entry Point Pattern by Complexity

**Single entry point** — for simple packages (most cases):

```typescript
// src/index.ts
export { getProducts } from './get-products'
export { createOrder } from './create-order'
export type { GetProductsArgs, GetProductsResponse } from './get-products'
```

**Multiple entry points** — for packages with distinct consumer groups (e.g., client vs server):

```json
{
  "exports": {
    ".": { "types": "./src/index.ts", "import": "./dist/index.js" },
    "./server": { "types": "./src/server/index.ts", "import": "./dist/server/index.js" }
  }
}
```

Why: Single entry point simplifies imports. Multiple entry points enable tree-shaking when consumers only need a subset.

---

## Use Workspace References for Internal Dependencies

When packages depend on other internal packages, use workspace protocol:

```json
{
  "dependencies": {
    "@scope/errors": "workspace:*",
    "@scope/types": "workspace:*"
  }
}
```

Why: Automatic version resolution within the workspace. Changes propagate immediately during development without publishing.

---

## Build Client Packages as Thin Shims

API client packages are **thin shims** — they build requests, normalize responses, and provide types. Nothing more.

**A client package should:**

- Build requests (URL construction, headers, params)
- Normalize responses (snake_case to camelCase, field mapping)
- Provide types for the API contract
- Validate responses with Zod at the boundary

**A client package should NOT:**

- Add business logic or abstractions beyond the API
- Include convenience utilities that filter or transform responses
- Combine multiple API calls into one function
- Manage caching or state (that's React Query's job)

Why: Thin shims keep the API contract clear. Consumers understand exactly what API capabilities they're using.

---

## Export Executable Endpoint Functions

Each API endpoint maps to one exported function. The function handles the full request lifecycle:

```typescript
export async function getProducts(
  config: GetProductsConfig,
  args: GetProductsArgs,
  fetchFn: typeof fetch = fetch,
): Promise<GetProductsResponse> {
  const url = new URL(`${config.baseUrl}/products/${args.market}`)
  url.searchParams.set('locale', args.locale)

  const res = await fetchFn(url.toString())
  if (!res.ok) {
    throw await res.json()
  }
  return GetProductsResponseSchema.parse(await res.json())
}
```

**Function signature pattern:** `endpoint(config, args, fetchFn = fetch)`

- `config` first — environment values (baseUrl, API keys) that rarely change
- `args` second — request-specific values (market, locale) that vary per call
- `fetchFn` last — injectable for logging or testing

Why: One call site, one place for response validation, consistent error shape. The injectable `fetchFn` enables server-side logging (e.g., `loggedFetch`) without coupling to implementation.

---

## Name Types by Function Coupling

Types are named after the function they belong to:

| Type             | Naming Pattern           | Example               |
| ---------------- | ------------------------ | --------------------- |
| Request args     | `{FunctionName}Args`     | `GetProductsArgs`     |
| Environment config | `{FunctionName}Config` | `GetProductsConfig`   |
| Response         | `{FunctionName}Response` | `GetProductsResponse` |

**Args vs Config:**

- `Args` — request-specific values that vary per call (market, locale, filters)
- `Config` — environment values that stay constant (baseUrl, API keys)

Why: Consistent naming makes types discoverable. Separating config from args clarifies what changes per call vs what's fixed per environment.

---

## Colocate Types with Function Files

Declare all types in the same file as the function they serve:

```
packages/hydra-client/src/
  get-customer.ts      # Contains GetCustomerArgs, GetCustomerConfig, GetCustomerResponse + function
  create-order.ts      # Contains CreateOrderArgs, CreateOrderConfig, CreateOrderResponse + function
  types/
    errors.ts          # Shared error types (one per client package)
```

- Function-specific types live in the function file
- Shared types (error shapes) go in `types/` subdirectory
- Re-export everything from `src/index.ts`

Why: Colocation eliminates hunting across files. Developers find the type right next to the function that uses it.

---

## Avoid Filtering Utilities in Client Packages

**Don't** export convenience functions that filter or transform API responses:

```typescript
// BAD: Don't do this in the client package
export function findProductBySku(
  response: GetProductsResponse,
  sku: string,
): Product | undefined {
  return response.items.find((item) => item.variations.some((v) => v.sku === sku))
}

// GOOD: Let consumers filter in their own code
const { data } = useGetProductsQuery({ market, locale })
const product = data?.items.find((item) => item.variations.some((v) => v.sku === sku))
```

Why: Filtering utilities obscure API capabilities, imply server-side filtering that doesn't exist, and hide the real API contract from consumers.

---

## Export Only What the API Provides

Don't create derived types in the client package. Consumers use indexed access types when they need sub-types:

```typescript
// Consumer code — derive from API response
import type { GetProductsResponse } from '@scope/cosmic-client'

type Product = GetProductsResponse['items'][number]
type Policy = NonNullable<Product['policies']>[number]
```

Why: Origin of the type is explicit. Client package stays thin. Less maintenance when API types change.
