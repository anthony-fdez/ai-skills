---
name: managing-state
description: Enforces Zustand state management patterns including store structure, actions, and subscriptions. Use when working with global state, creating Zustand stores or slices, adding actions, using selectors, or any time code might sync React Query data into Zustand (a critical anti-pattern to prevent). Also trigger when optimizing re-renders, deciding whether data belongs in Zustand vs React Query, or when a component subscribes to the full store instead of specific slices.
---

# Managing State (Zustand)

Zustand is for **CLIENT state only**. See `writing-react-query` skill for server/API state.

## Contents

- [Never Sync React Query Data to Zustand](#never-sync-react-query-to-zustand)
- [Use Single Global Store Architecture](#store-architecture)
- [Use Partial Updates with Spread](#action-patterns)
- [Subscribe Selectively to Needed State](#selective-subscriptions)
- [Know What Belongs Where](#what-belongs-where)

---

## NEVER Sync React Query to Zustand

```typescript
// ❌ BAD: Syncing React Query data or loading states to Zustand
useEffect(() => {
  if (items) {
    setForm({ selectedItem: defaultItem })
  }
}, [items])

useEffect(() => {
  setUI({ isLoadingItems: isLoading })
}, [isLoading])

// ✅ GOOD: Keep API data ONLY in React Query, derive during render
const { data: items, isLoading } = useGetItemsQuery()
const defaultItem = items?.find((item) => item.isDefault) ?? null

return <ItemSelector items={items} default={defaultItem} />
```

## Store Architecture

```typescript
// store/useGlobalStore.ts — single global store
export const useGlobalStore = create<GlobalStore>()(
  devtools(
    persist(
      (set, get) => ({
        ui: uiInitialValues,
        cart: cartInitialValues,
        preferences: preferencesInitialValues,
        setUI: (ui) => set((state) => ({ ui: { ...state.ui, ...ui } })),
      }),
      { name: 'global-store', partialize: (state) => ({ cart: state.cart }) },
    ),
  ),
)
```

## Action Patterns

```typescript
// Partial updates with spread (preferred)
setUI({ isModalOpen: true })
setPreferences({ theme: 'dark' })

// Collection actions (client state — user's selection)
addToCart: (item: CartItemType) => {
  set((state) => {
    const existing = state.cart.items.find((i) => i.id === item.id)
    if (existing) {
      return { cart: { ...state.cart, items: state.cart.items.map((i) =>
        i.id === item.id ? { ...i, quantity: i.quantity + item.quantity } : i
      )}}
    }
    return { cart: { ...state.cart, items: [...state.cart.items, item] } }
  })
}
```

## Selective Subscriptions

```typescript
// ✅ Good: Subscribe only to what you need
const isModalOpen = useGlobalStore((state) => state.ui.isModalOpen)

// ✅ Good: Multiple values with shallow
const { cart, setCart } = useGlobalStore(
  (state) => ({ cart: state.cart, setCart: state.setCart }),
  shallow,
)

// ✅ Good: Compute in selector, don't store derived state
const cartTotal = useGlobalStore((state) =>
  state.cart.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
)

// ❌ Bad: Full store (re-renders on ANY change)
const store = useGlobalStore()
```

## What Belongs Where

| Data Type        | Where       | Example                        |
| ---------------- | ----------- | ------------------------------ |
| UI state         | Zustand     | `isModalOpen`, `isSidebarOpen` |
| User preferences | Zustand     | `theme`, `locale`              |
| Auth tokens      | Zustand     | `token` (needed for requests)  |
| Cart items       | Zustand     | User's product selections      |
| User profile     | React Query | `useGetUserQuery()`            |
| Items/Products   | React Query | `useGetItemsQuery()`           |
| Settings         | React Query | `useGetSettingsQuery()`        |
| Orders           | React Query | `useGetOrdersQuery()`          |

## Performance Tips

1. Subscribe to only needed state — prevents re-renders
2. Use `shallow` for object selections
3. Compute derived values in selectors — don't store them
4. Keep actions pure — handle async in hooks that call store actions

## Checklist

- [ ] API data is in React Query, NOT Zustand
- [ ] No useEffect syncing React Query data or loading states to Zustand
- [ ] Components subscribe only to needed state (shallow for objects)
- [ ] Derived values computed in selectors (not stored)
- [ ] Async operations in hooks, not in actions
