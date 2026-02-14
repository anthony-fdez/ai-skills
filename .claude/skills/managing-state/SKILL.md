---
name: managing-state
description: Enforces Zustand state management patterns including store structure, actions, and subscriptions. Use when working with global state, creating actions, or optimizing re-renders.
---

# Managing State (Zustand)

Patterns for Zustand state management. **Critical:** Zustand is for CLIENT state only. See `writing-react-query` skill for server/API state.

## The Golden Rule: Server State vs Client State

### Zustand is for CLIENT STATE ONLY

```typescript
// ✅ Zustand: UI state, preferences, tokens
type AppType = {
  isCartOpen: boolean
  isLoginPopoverOpen: boolean
  isSearchBarActive: boolean
  selectedLocale: string
  theme: 'light' | 'dark'
  token: string | null // Auth token needed for requests
}

// ❌ NEVER put API data in Zustand
type UserType = {
  customer: CustomerObjectType // ❌ Use React Query
  creditCardsData: CreditCardsData // ❌ Use React Query
  shippingAddresses: AddressType[] // ❌ Use React Query
}
```

### React Query is for SERVER STATE

```typescript
// ✅ Good: API data lives in React Query
const { data: customer } = useGetCustomerQuery()
const { data: shippingAddresses } = useGetShippingAddressesQuery()
const { data: creditCards } = useGetCreditCardsQuery()
```

## NEVER Sync React Query to Zustand

### This is an Anti-Pattern

```typescript
// ❌ BAD: Syncing React Query data to Zustand with useEffect
const useGetShippingAddressesQuery = () => {
  const { setOrder } = useGlobalStore()

  const { data: shipToOptions } = useQuery({
    queryKey: ['shipping-addresses'],
    queryFn: fetchAddresses,
  })

  // ❌ ANTI-PATTERN: Multiple sources of truth!
  useEffect(() => {
    if (shipToOptions) {
      const defaultAddress = shipToOptions.find((a) => a.isDefault)
      setOrder({ shipping: defaultAddress })
    }
  }, [shipToOptions])
}
```

### The Correct Pattern

```typescript
// ✅ GOOD: Keep API data ONLY in React Query
const useGetShippingAddressesQuery = () => {
  const { data: shipToOptions, isLoading } = useQuery({
    queryKey: ['shipping-addresses'],
    queryFn: fetchAddresses,
  })

  // Derive values during render - no state, no useEffect
  const defaultAddress = shipToOptions?.find((a) => a.isDefault) ?? null

  return { shipToOptions, defaultAddress, isLoading }
}

// Use directly in component
const CheckoutShipping = () => {
  const { shipToOptions, defaultAddress } = useGetShippingAddressesQuery()
  return <AddressSelector addresses={shipToOptions} default={defaultAddress} />
}
```

## NEVER Sync Loading States to Zustand

```typescript
// ❌ BAD: Syncing React Query loading state to Zustand
useEffect(() => {
  setLoaders({ isLoadingSavedAddresses: isLoadingShipToOptions })
}, [isLoadingShipToOptions])

// ✅ GOOD: Use loading state directly from React Query
const { isLoading } = useGetShippingAddressesQuery()

if (isLoading) return <Spinner />
```

## Store Architecture

### Single Global Store

```typescript
// store/useGlobalStore.ts
export const useGlobalStore = create<GlobalStore>()(
  devtools(
    persist(
      (set, get) => ({
        // CLIENT state only
        app: appInitialValues,
        cart: cartInitialValues, // Cart items (client-side)
        checkout: checkoutInitialValues, // Checkout UI state
        loaders: loadersInitialValues, // UI loading states (not API loading)
        // Actions
        setApp: (app) => set((state) => ({ app: { ...state.app, ...app } })),
      }),
      {
        name: 'global-store',
        partialize: (state) => ({
          cart: state.cart,
        }),
      },
    ),
  ),
)
```

## Action Patterns

### Partial Updates (Preferred)

```typescript
// ✅ Good: Partial updates with spread
setApp: (app: Partial<AppType>) => {
  set((state) => ({
    app: { ...state.app, ...app },
  }))
}

// Usage
setApp({ isCartOpen: true })
```

### Cart Actions (Client State)

```typescript
// Cart items are client state (user's selection)
addToCart: (item: CartItemType) => {
  set((state) => {
    const existing = state.cart.items.find((i) => i.id === item.id)

    if (existing) {
      return {
        cart: {
          ...state.cart,
          items: state.cart.items.map((i) =>
            i.id === item.id
              ? { ...i, quantity: i.quantity + item.quantity }
              : i,
          ),
        },
      }
    }

    return {
      cart: { ...state.cart, items: [...state.cart.items, item] },
    }
  })
}
```

## Selective Subscriptions

### Subscribe Only to What You Need

```typescript
// ✅ Good: Selective subscription
const isCartOpen = useGlobalStore((state) => state.app.isCartOpen)

// ✅ Good: Multiple values with shallow
import { shallow } from 'zustand/shallow'

const { cart, setCart } = useGlobalStore(
  (state) => ({ cart: state.cart, setCart: state.setCart }),
  shallow,
)

// ❌ Bad: Full store (re-renders on ANY change)
const store = useGlobalStore()
```

## Computed Values (Selectors)

```typescript
// ✅ Good: Compute in selector, don't store
const cartTotal = useGlobalStore((state) =>
  state.cart.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
)

// ❌ Bad: Storing derived state
type CartType = {
  items: CartItemType[]
  total: number // Don't store! Compute from items
}
```

## What Belongs Where

| Data Type          | Where       | Example                          |
| ------------------ | ----------- | -------------------------------- |
| UI state           | Zustand     | `isCartOpen`, `isModalOpen`      |
| User preferences   | Zustand     | `theme`, `locale`                |
| Auth tokens        | Zustand     | `token` (needed for requests)    |
| Cart items         | Zustand     | User's product selections        |
| Customer data      | React Query | `useGetCustomerQuery()`          |
| Shipping addresses | React Query | `useGetShippingAddressesQuery()` |
| Payment methods    | React Query | `useGetPaymentMethodsQuery()`    |
| Products           | React Query | `useGetProductsQuery()`          |
| Orders             | React Query | `useGetOrdersQuery()`            |

## Reset Functionality

```typescript
resetCart: () => {
  set({ cart: cartInitialValues })
}

resetCheckout: () => {
  set({ checkout: checkoutInitialValues })
}
```

## Performance Tips

1. **Subscribe to only needed state** - prevents re-renders
2. **Use shallow** for object selections
3. **Compute derived values in selectors** - don't store them
4. **Keep actions pure** - no async inside actions
5. **Handle async in hooks** that call store actions

```typescript
// ✅ Good: Async in hook, action is pure
const useCheckout = () => {
  const setCheckout = useGlobalStore((state) => state.setCheckout)

  const processPayment = async (data: PaymentData) => {
    setCheckout({ isProcessing: true })
    try {
      const result = await paymentService.process(data)
      setCheckout({ isProcessing: false })
    } catch (error) {
      setCheckout({ isProcessing: false, error: error.message })
    }
  }

  return { processPayment }
}
```

## Checklist

- [ ] API data is in React Query, NOT Zustand
- [ ] No useEffect syncing React Query data to Zustand
- [ ] No useEffect syncing loading states to Zustand
- [ ] Zustand only contains client state (UI, preferences, tokens)
- [ ] Components subscribe only to needed state
- [ ] shallow used for object selections
- [ ] Derived values computed in selectors (not stored)
- [ ] Async operations in hooks, not in actions

## Related Skills

- See `writing-react-query` for server state patterns
- See `writing-react` for state anti-patterns (useEffect chains, duplicated state)
