---
name: writing-react
description: Enforces React component patterns, state management, and anti-patterns specific to this codebase. Use when writing any React component, handling component state, using hooks, passing props, structuring JSX, or refactoring component logic. Also trigger when reviewing components for anti-patterns like syncing props to state via useEffect, using useEffect for event handling or data fetching, creating render functions instead of extracting components, or passing setters as props instead of callbacks.
---

# Writing React Components

## Contents

- [Structure Components as Hooks → Handlers → Render](#component-structure)
- [Pass Callbacks, Not Setters](#props-pass-callbacks-not-setters)
- [Don't Sync Props to State](#dont-sync-props-to-state)
- [Don't Store Computed Values](#dont-store-computed-values)
- [Don't Use useEffect for Events](#no-useeffect-for-events--track-in-the-handler)
- [Don't Use useEffect for Data Fetching](#no-useeffect-for-data-fetching--use-react-query)
- [Extract Complex Conditionals to Named Booleans](#extract-complex-conditionals)
- [Extract Components, Not Render Functions](#extract-components-not-render-functions)
- [Keep API Data in React Query Only](#data-flow-single-source-of-truth)

---

## Component Structure

```tsx
const ProductCard = ({ product, onAddToCart }: ProductCardProps) => {
  // 1. Hooks first
  const [isExpanded, setIsExpanded] = useState(false)
  const { data: reviews } = useGetReviewsQuery({ productId: product.id })

  // 2. Event handlers
  const handleAddToCart = () => {
    onAddToCart(product.id)
  }

  // 3. Render
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <Button onClick={handleAddToCart}>Add to Cart</Button>
    </div>
  )
}
```

Arrow function components only. No `function` declarations.

## Props: Pass Callbacks, Not Setters

```tsx
// ❌ Bad
<QuantityControls itemCount={itemCount} setItemCount={setItemCount} />

// ✅ Good
<QuantityControls itemCount={itemCount} onItemCountChange={handleItemCountChange} />
```

Why: Setters expose implementation details and couple the child to the parent's state shape. Callbacks let the parent control what happens, making the child reusable.

## State Anti-Patterns

### Don't Sync Props to State

```tsx
// ❌ Bad: useState + useEffect to sync
const [selectedItem, setSelectedItem] = useState(getDefault(items, query))
useEffect(() => { setSelectedItem(getDefault(items, query)) }, [query.id])

// ✅ Good: Derive during render
const selectedItem = useMemo(
  () => getDefault(items, query),
  [items, query.id],
)
```

Why: useState + useEffect sync creates a render where the old value is visible before the effect runs. Deriving during render is always in sync.

### Don't Store Computed Values

```tsx
// ❌ Bad
const [hasMultipleItems, setHasMultipleItems] = useState(false)
useEffect(() => { setHasMultipleItems(items?.length > 1) }, [items])

// ✅ Good
const hasMultipleItems = (items?.length ?? 0) > 1
```

## useEffect Anti-Patterns

### No useEffect for Events — Track in the Handler

```tsx
// ❌ Bad
useEffect(() => { if (isModalOpen) analytics.track('modal_viewed') }, [isModalOpen])

// ✅ Good
const handleOpenModal = () => {
  setUI({ isModalOpen: true })
  analytics.track('modal_viewed')
}
```

### No useEffect for Data Fetching — Use React Query

```tsx
// ❌ Bad
const [data, setData] = useState(null)
useEffect(() => { fetchData().then(setData) }, [deps])

// ✅ Good
const { data, isLoading } = useQuery({ queryKey: ['data', deps], queryFn: fetchData })
```

## JSX Patterns

### Extract Complex Conditionals

```tsx
// ❌ Bad: Logic buried in JSX
{!isFocusMode && !isLoading && !isCompact && willShowSuggestions && <RelatedContent />}

// ✅ Good: Named boolean
const shouldShowSuggestions = !isFocusMode && !isLoading && !isCompact && willShowSuggestions
{shouldShowSuggestions && <RelatedContent />}
```

### Extract Components, Not Render Functions

```tsx
// ❌ Bad: Render function recreated every render
const renderDescription = () => { ... }
return <div>{renderDescription()}</div>

// ✅ Good: Separate component
<ProductDescription isCompact={isCompact} item={selectedItem} />
```

Why: Render functions are recreated every render and can't have their own hooks. Separate components get their own lifecycle and memoization boundary.

### No Inline Complex Logic

Extract handlers longer than a few lines into named functions.

## Context: Don't Overuse

```tsx
// ❌ Bad: Context with frequently changing values — use Zustand instead
// ❌ Bad: Context for 2-level prop passing — just pass props

// ✅ Good: Zustand for frequently changing state (components subscribe to only what they need)
// ✅ Good: Composition pattern to avoid prop drilling
```

Why: Context re-renders all consumers on any change. Zustand with selectors lets components subscribe to only the state they need.

## Data Flow: Single Source of Truth

API data stays in React Query — never sync to Zustand or useState.

```tsx
const { data: items } = useGetItemsQuery()
const defaultItem = items?.find((item) => item.isDefault) ?? null
return <ItemSelector items={items} default={defaultItem} />
```

## Checklist

- [ ] Hooks → handlers → render order
- [ ] Props use `onX` naming for callbacks (not setters)
- [ ] Derived state computed, not stored in useState
- [ ] No useEffect for event handling or data fetching
- [ ] Complex conditionals extracted to named booleans
- [ ] Components extracted (not render functions)
- [ ] API data stays in React Query only
- [ ] `data-test` attributes added for E2E testing
