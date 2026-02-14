---
name: organizing-files
description: Enforces flat file structure with descriptive naming. Use when creating files, renaming files, moving files, reorganizing code, refactoring folder structure, or reviewing file organization. Applies to components, hooks, functions, services, and utilities.
---

# File Structure & Organization

## Core Principles

1. **Flat over nested** - Avoid arbitrary grouping folders
2. **Self-describing names** - Files should not rely on folder context
3. **Consistency** - Same patterns across the entire codebase

## Naming Conventions

### Components (`.tsx`)

**Format**: `PascalCase.tsx` - NOT `folder/index.tsx`

```
ProductCard.tsx           ✓
CheckoutForm.tsx          ✓
product-card/index.tsx    ✗
checkout/Form.tsx         ✗
```

### Hooks & Functions (`.ts`)

**Format**: `kebab-case.ts` with full context in filename

```
use-product-pricing.ts              ✓
get-customer-subscriptions.ts       ✓
format-currency-amount.ts           ✓
usePricing.ts                       ✗ (not descriptive)
utils/format.ts                     ✗ (relies on folder)
```

### Services (`.ts`)

**Format**: `{service}-{action}-{resource}.ts`

```
cosmic-get-all-products.ts          ✓
hydra-update-customer-cart.ts       ✓
mixpanel-track-purchase-event.ts    ✓
cosmic/products/get-all.ts          ✗ (nested)
get-products.ts                     ✗ (missing service context)
```

### Types (`.ts` or `.d.ts`)

**Format**: `{domain}-{name}.types.ts`

```
cosmic-product.types.ts             ✓
hydra-customer.types.ts             ✓
types/product.ts                    ✗
```

### Constants & Config

**Format**: `{domain}-{purpose}.constants.ts`

```
checkout-shipping-methods.constants.ts    ✓
product-categories.constants.ts           ✓
constants.ts                              ✗
```

## Folder Structure

### Folder Naming

**Format**: `kebab-case/` for all folders

```text
address-restrictions/         ✓
addressRestrictions/          ✗ (camelCase)
AddressRestrictions/          ✗ (PascalCase)
```

### Acceptable Grouping

- Feature domains: `components/`, `lib/`, `pages/`
- Component-specific helpers used ONLY by that component
- Co-located test files
- Domain subfolders for related items (e.g., `address-restrictions/restrictions/` for individual restriction files)

### Domain Subfolder Pattern

When a domain has many related items of the same type, use a subfolder:

```text
lib/utils/address-restrictions/
├── address-restriction.types.ts          # Types at root
├── create-address-restriction.ts         # Factory at root
├── get-all-address-restrictions.ts       # Aggregator at root
└── restrictions/                         # Related items in subfolder
    ├── united-states-to-puerto-rico.ts
    ├── po-box-restriction.ts
    └── military-address.ts
```

### Avoid

- Deeply nested folders: `services/cosmic/products/helpers/`
- Generic groupings: `utils/helpers/`, `common/shared/`
- Type-only folders: `types/`, `interfaces/`
- camelCase or PascalCase folder names

## Quick Reference

| Type      | Pattern                            | Example                      |
| --------- | ---------------------------------- | ---------------------------- |
| Component | `PascalCase.tsx`                   | `ProductCard.tsx`            |
| Hook      | `use-{descriptive-name}.ts`        | `use-cart-totals.ts`         |
| Function  | `{action}-{resource}.ts`           | `format-price-display.ts`    |
| Service   | `{service}-{action}-{resource}.ts` | `cosmic-get-product-data.ts` |
| Type      | `{domain}.types.ts`                | `checkout.types.ts`          |
| Constant  | `{domain}.constants.ts`            | `shipping.constants.ts`      |

## Refactoring Steps

1. Identify files with non-descriptive names
2. Rename with full context (include service/domain prefix)
3. Flatten unnecessary folder nesting
4. Update all import paths
5. Verify: `npx tsc --noEmit`
6. Lint: `npm run formatAndLint`

## Anti-Patterns

### Nested Folder Structure

```bash
# ❌ Never do this
components/
  Button/
    index.tsx           → Button.tsx
    styles.ts           → button-styles.ts

services/
  cosmic/
    products/
      get-all.ts        → cosmic-get-all-products.ts
      get-by-id.ts      → cosmic-get-product-by-id.ts

utils/
  helpers/
    format.ts           → format-{specific-thing}.ts
```

### Multiple Exports Per File

Each file should export exactly one function. This makes files easier to find, test, and maintain.

```typescript
// ❌ Bad: Multiple exports in one file
// utils/format.ts
export const formatPrice = (price: number) => `$${price.toFixed(2)}`
export const formatDate = (date: Date) => date.toISOString()
export const formatName = (first: string, last: string) => `${first} ${last}`

// ✅ Good: One function per file
// format-price-display.ts
export const formatPriceDisplay = (price: number) => `$${price.toFixed(2)}`

// format-date-iso.ts
export const formatDateIso = (date: Date) => date.toISOString()

// format-full-name.ts
export const formatFullName = (first: string, last: string) =>
  `${first} ${last}`
```

### Tight Coupling (Hardcoded Dependencies)

Pass dependencies as props instead of importing them directly. This makes components testable and reusable.

```typescript
// ❌ Bad: Tightly coupled to specific service
import { fetchProducts } from '@/lib/services/cosmic-get-products'

const ProductList = () => {
  const [products, setProducts] = useState([])

  useEffect(() => {
    fetchProducts().then(setProducts)  // Hardcoded dependency
  }, [])

  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>
}

// ✅ Good: Dependencies passed as props (or via React Query hooks)
type ProductListProps = {
  products: Product[]
}

const ProductList = ({ products }: ProductListProps) => {
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>
}

// Parent component handles data fetching
const ProductPage = () => {
  const { data: products } = useGetProductsQuery()
  return <ProductList products={products ?? []} />
}
```

### Legacy Terminology

Use descriptive names that reflect actual purpose/format, not "legacy" or "old" labels.

```typescript
// ❌ Bad: Legacy terminology
const getLegacyOrderShape = () => ({ items: [{ id: 1 }] })
const getOrderShape = () => ({ items: { id: 1 } })

// ✅ Good: Descriptive naming
const getOrderShapeWithItemsInArray = () => ({ items: [{ id: 1 }] })
const getOrderShapeWithItemsInObject = () => ({ items: { id: 1 } })
```

## Checklist

- [ ] Files have self-describing names (no reliance on folder context)
- [ ] One function/component per file
- [ ] No nested folder structures for services/utils
- [ ] Components use PascalCase.tsx (not folder/index.tsx)
- [ ] Hooks/functions use kebab-case with full context
- [ ] Services include service prefix (cosmic-, hydra-, etc.)
- [ ] Dependencies passed as props, not hardcoded imports
- [ ] No "legacy" or "old" in naming - use descriptive names
