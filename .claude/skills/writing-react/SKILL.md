---
name: writing-react
description: Enforces React component patterns, state management, and anti-patterns. Use when writing any React component, handling state, or using hooks.
---

# Writing React Components

Patterns for writing React components including structure, state management, and common anti-patterns to avoid.

## Component Structure

### Standard Component Pattern

```tsx
// ✅ Good: Proper component structure
const ProductCard = ({ product, onAddToCart }: ProductCardProps) => {
  // 1. Hooks first
  const translate = useTranslate()
  const [isExpanded, setIsExpanded] = useState(false)

  // 2. Event handlers
  const handleAddToCart = () => {
    onAddToCart(product.id)
  }

  // 3. Render
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <Button onClick={handleAddToCart}>{translate('add_to_cart')}</Button>
    </div>
  )
}
```

### Arrow Functions for Components

```tsx
// ✅ Good: Arrow function component
const UserProfile = ({ user }: UserProfileProps) => {
  return <div>{user.name}</div>
}

// ❌ Bad: Function declaration
function UserProfile({ user }: UserProfileProps) {
  return <div>{user.name}</div>
}
```

## Props Patterns

### Standard Props Pattern

```tsx
// ✅ Good: Props with proper naming
type ButtonProps = {
  className?: string // Always optional
  children?: React.ReactNode
  onPress?: () => void // Callbacks prefixed with 'on'
  isLoading?: boolean
}
```

### Pass Callbacks, Not Setters

```tsx
// ❌ Bad: Passing state setters directly
<QuantityControls
  itemCount={itemCount}
  setItemCount={setItemCount}  // Direct setter
/>

// ✅ Good: Pass named callbacks
<QuantityControls
  itemCount={itemCount}
  onItemCountChange={handleItemCountChange}  // Named callback
/>

// Parent controls the logic
const handleItemCountChange = (newCount: number) => {
  if (newCount < 1) return
  if (newCount > maxCount) return
  trackQuantityChange(newCount)
  setItemCount(newCount)
}
```

## State Management Anti-Patterns

### Duplicated State - Don't Sync Props to State

```tsx
// ❌ Bad: Storing props in state and syncing with useEffect
const ProductDetails = ({ productData, query }: Props) => {
  const [selectedVariation, setSelectedVariation] = useState(
    getDefaultVariation(productData.variations, query),
  )

  useEffect(() => {
    setSelectedVariation(getDefaultVariation(productData.variations, query))
  }, [query.sku, productData.variations])
}

// ✅ Good: Derive during render
const ProductDetails = ({ productData, query }: Props) => {
  const selectedVariation = useMemo(
    () => getDefaultVariation(productData.variations, query),
    [productData.variations, query.sku],
  )
}
```

### Storing Computed State

```tsx
// ❌ Bad: Storing derived values in state
const [hasMultipleVariations, setHasMultipleVariations] = useState(false)

useEffect(() => {
  if (variations?.length > 1) {
    setHasMultipleVariations(true)
  }
}, [variations])

// ✅ Good: Compute during render
const hasMultipleVariations = (variations?.length ?? 0) > 1
```

## useEffect Anti-Patterns

### Avoid useEffect for Event Handlers

```tsx
// ❌ Bad: useEffect to track when cart opens
useEffect(() => {
  if (isCartOpen) {
    gtm.viewCart(currentItems)
    mixpanelService.trackEvent(MixpanelEvent.VIEW_CART, { ... })
  }
}, [isCartOpen])

// ✅ Good: Track in the event handler
const handleOpenCart = () => {
  setApp({ isCartOpen: true })
  gtm.viewCart(currentItems)
  mixpanelService.trackEvent(MixpanelEvent.VIEW_CART, { ... })
}
```

### Avoid useEffect Chains

```tsx
// ❌ Bad: State changes triggering more state changes
useEffect(() => {
  setHasTrackedViewProduct(false)
}, [productData.slug])

// ✅ Good: Use refs for tracking
const hasTrackedProductRef = useRef(false)
const lastTrackedSlugRef = useRef<string | null>(null)

useEffect(() => {
  if (lastTrackedSlugRef.current !== productData.slug) {
    hasTrackedProductRef.current = false
    lastTrackedSlugRef.current = productData.slug
  }

  if (hasTrackedProductRef.current) return
  hasTrackedProductRef.current = true
}, [productData.slug])
```

### Avoid useEffect for Data Fetching

```tsx
// ❌ Bad: Manual data fetching with useEffect
const [data, setData] = useState(null)
const [isLoading, setIsLoading] = useState(false)

useEffect(() => {
  setIsLoading(true)
  fetchData()
    .then(setData)
    .finally(() => setIsLoading(false))
}, [deps])

// ✅ Good: Use React Query
const { data, isLoading } = useQuery({
  queryKey: ['data', deps],
  queryFn: fetchData,
})
```

## JSX Patterns

### Extract Complex Conditionals

```tsx
// ❌ Bad: Complex inline conditionals
{
  !isFocusMode &&
    !isLoadingIsFocusMode &&
    !isQuickAdd &&
    willShowUnlockedProductFlow && (
      <FrequentlyBoughtTogether productData={productData} />
    )
}

// ✅ Good: Named boolean
const shouldShowFrequentlyBought =
  !isFocusMode &&
  !isLoadingIsFocusMode &&
  !isQuickAdd &&
  willShowUnlockedProductFlow

{
  shouldShowFrequentlyBought && (
    <FrequentlyBoughtTogether productData={productData} />
  )
}
```

### Avoid Render Functions Inside Components

```tsx
// ❌ Bad: Render functions recreated every render
const ProductDetails = () => {
  const renderDescription = () => {
    if (isQuickAdd) return <QuickAddView />
    return <FullDescription />
  }

  return <div>{renderDescription()}</div>
}

// ✅ Good: Extract to component
const ProductDescription = ({ isQuickAdd, selectedVariation }: Props) => {
  if (isQuickAdd) return <QuickAddView />
  return <FullDescription variation={selectedVariation} />
}

const ProductDetails = () => {
  return (
    <ProductDescription
      isQuickAdd={isQuickAdd}
      selectedVariation={selectedVariation}
    />
  )
}
```

### Complex Logic in JSX

```tsx
// ❌ Bad: 70+ lines of logic in onClick
;<AddToCartButton
  onClick={() => {
    if (
      itemToAddToCart.po === 'subscribe' &&
      addProductToSubscription.userHasActiveSubscription
    ) {
      setAppAddProductToSubscription({
        isSelectActiveSubscriptionModalOpen: true,
      })
      return
    }
    const eventLocation = isQuickAdd
      ? 'quick_add_page_main_button'
      : 'product_page_main_button'
    gtm.addToCart({ cartItem: itemToAddToCart, event_location: eventLocation })
    // ... 50 more lines
  }}
>
  <T>add_to_cart</T>
</AddToCartButton>

// ✅ Good: Extract to named handler
const handleAddToCart = () => {
  if (shouldShowSubscriptionModal()) {
    setAppAddProductToSubscription({
      isSelectActiveSubscriptionModalOpen: true,
    })
    return
  }
  const eventLocation = getEventLocation()
  trackAddToCart(eventLocation)
  addItemToCart(eventLocation)
}

;<AddToCartButton onClick={handleAddToCart}>
  <T>add_to_cart</T>
</AddToCartButton>
```

## Context Anti-Patterns

### Don't Overuse Context

```tsx
// ❌ Bad: Context with frequently changing values
type NavigationMenuContextType = {
  isNavigationMenuOpen: boolean // Changes on open/close
  navigationHeight: number // Changes on resize
  navigationMenuValue: string // Changes on tab selection
  offsetHeight: number // Changes on scroll/resize
}

// ✅ Good: Use Zustand for frequently changing state
const useNavigationStore = create((set) => ({
  isNavigationMenuOpen: false,
  navigationMenuValue: '',
  setIsNavigationMenuOpen: (open) => set({ isNavigationMenuOpen: open }),
}))

// Components subscribe to only what they need
const MenuButton = () => {
  const isOpen = useNavigationStore((state) => state.isNavigationMenuOpen)
  return <button onClick={() => setIsOpen(!isOpen)} />
}
```

### Context is Not for Prop Drilling (2-3 levels)

```tsx
// ❌ Bad: Context for 2-level prop passing
const FormStateContext = createContext()

const GrandParent = () => (
  <FormStateContextProvider>
    <Parent />
  </FormStateContextProvider>
)

// ✅ Good: Just pass props (2 levels is fine)
const GrandParent = () => {
  const [formValue, setFormValue] = useState('')
  return <Parent formValue={formValue} />
}

const Parent = ({ formValue }) => <Child formValue={formValue} />

// ✅ Better: Use composition
const GrandParent = () => {
  const [formValue, setFormValue] = useState('')
  return (
    <Parent>
      <Child formValue={formValue} />
    </Parent>
  )
}
```

## UI Component Patterns

### Variant System with CVA

```tsx
// ✅ Good: CVA for variants
const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground',
        outline: 'border border-input bg-background',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  },
)

const Button = ({ className, variant, size, ...props }: ButtonProps) => (
  <button
    className={cn(buttonVariants({ variant, size }), className)}
    {...props}
  />
)

export { Button, buttonVariants }
```

### Compound Components

```tsx
// ✅ Good: Compound component pattern
<Dialog>
  <DialogTrigger asChild>
    <Button>Open</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Title</DialogTitle>
      <DialogDescription>Description</DialogDescription>
    </DialogHeader>
  </DialogContent>
</Dialog>
```

### Forward Refs for Form Components

```tsx
// ✅ Good: Forward refs for inputs
const Input = React.forwardRef<
  HTMLInputElement,
  React.InputHTMLAttributes<HTMLInputElement>
>(({ className, type, ...props }, ref) => {
  return (
    <input
      type={type}
      className={cn(
        'flex h-10 w-full rounded-md border border-input bg-background px-3 py-2',
        className,
      )}
      ref={ref}
      {...props}
    />
  )
})
Input.displayName = 'Input'
```

## Accessibility

### Required Patterns

```tsx
// ✅ Good: Accessible component
<button
  aria-label="Close dialog"
  onClick={onClose}
  className={cn(
    'focus-visible:outline-none focus-visible:ring-2',
    'disabled:pointer-events-none disabled:opacity-50',
  )}
>
  <X className="h-4 w-4" />
</button>
```

## Data Flow

### Single Source of Truth

```tsx
// ❌ Bad: API data in multiple locations
// 1. React Query cache
// 2. Global Zustand store
useEffect(() => {
  if (shipToOptions) {
    setOrder({ shipping: defaultAddress }) // Syncing to store
  }
}, [shipToOptions])

// ✅ Good: Keep API data only in React Query
const { data: shipToOptions } = useGetShippingAddressesQuery()
const defaultAddress = shipToOptions?.find((a) => a.isDefault) ?? null

// Use directly in component
return <AddressSelector addresses={shipToOptions} default={defaultAddress} />
```

## Checklist

- [ ] Component uses arrow function syntax
- [ ] Hooks are declared first, then handlers, then render
- [ ] Props use `onX` naming for callbacks (not setters)
- [ ] Derived state is computed, not stored in useState
- [ ] No useEffect for event handling
- [ ] Complex conditionals extracted to named booleans
- [ ] Render logic extracted to separate components (not render functions)
- [ ] Context only used for stable, global data (not frequent updates)
- [ ] API data stays in React Query (not duplicated to global store)
- [ ] data-test attributes added for E2E testing
- [ ] ARIA labels and keyboard navigation supported
