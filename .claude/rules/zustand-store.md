---
description: Project-specific Zustand store structure and slice conventions
globs: ["src/store/**", "src/**/*.ts", "src/**/*.tsx"]
---

# Zustand Store Conventions

## Store Location

Single global store at `src/store/useGlobalStore.ts`.

## Available Slices

| Slice           | Purpose                           | CLIENT state? |
| --------------- | --------------------------------- | ------------- |
| `app`           | UI state (modals, locale, theme)  | Yes           |
| `cart`          | Cart items (user selections)      | Yes           |
| `cms`           | CMS configuration                 | Yes           |
| `user`          | Auth tokens, preferences          | Yes           |
| `order`         | Order UI state                    | Yes           |
| `subscriptions` | Subscription UI state             | Yes           |
| `checkout`      | Checkout UI state, processing     | Yes           |
| `loaders`       | UI loading states (NOT API loading) | Yes         |

## Actions Pattern

All setters use partial updates with spread:

```
setApp({ isCartOpen: true })
setCheckout({ isProcessing: false })
setCart({ items: [...] })
```

## Persistence

Only `cart` slice is persisted to localStorage via Zustand persist middleware.

## What Does NOT Belong Here

- Customer data (use React Query: `useGetCustomerQuery()`)
- Shipping addresses (use React Query: `useGetShippingAddressesQuery()`)
- Credit cards (use React Query: `useGetCreditCardsQuery()`)
- Products (use React Query: `useGetProductsQuery()`)
- Orders (use React Query: `useGetOrdersQuery()`)
- API loading states (use `isLoading` from React Query directly)
