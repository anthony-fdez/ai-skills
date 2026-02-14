---
name: writing-react-query
description: Enforces React Query patterns for data fetching, caching, mutations, and server state management. Use when writing queries, mutations, or handling API data.
---

# Writing React Query

Patterns for using React Query effectively, including hook organization, caching strategies, mutation patterns, and anti-patterns to avoid.

## Create Dedicated Query Hooks

### Don't Use useQuery Directly in Components

```tsx
// ❌ Bad: useQuery directly in component
const ProductList = () => {
  const { data, isLoading } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/api/products').then((r) => r.json()),
  })
}

const FeaturedProducts = () => {
  const { data, isLoading } = useQuery({
    queryKey: ['products'], // Duplicated configuration
    queryFn: () => fetch('/api/products').then((r) => r.json()),
  })
}

// ✅ Good: Create dedicated hook
// src/lib/hooks/queries/products/useGetProductsQuery.tsx
export const useGetProductsQuery = (filters?: ProductFilters) => {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: () => fetchProducts(filters),
    staleTime: 5 * 60 * 1000,
    gcTime: 10 * 60 * 1000,
    retry: 3,
  })
}

// Components just use the hook
const ProductList = () => {
  const { data: products, isLoading } = useGetProductsQuery({
    category: 'supplements',
  })
}

const FeaturedProducts = () => {
  const { data: products, isLoading } = useGetProductsQuery({ featured: true })
}
```

## Couple Params to Action Types

```tsx
// ✅ Good: Import types from the action/library
import type { CustomerByHrefArgs } from '@checkout/hydra-client/types'
import { useQuery } from '@tanstack/react-query'
import { getCustomerByHref } from '@/app/actions/hydra/get-customer-by-href.action'

export const useCustomer = ({ href }: CustomerByHrefArgs) => {
  return useQuery({
    queryKey: ['customer', href],
    queryFn: () => getCustomerByHref({ href }),
    enabled: !!href,
  })
}

// ❌ Bad: Duplicating types
type CustomerByHrefArgs = { href: string } // Don't duplicate!

export const useCustomer = ({ href }: CustomerByHrefArgs) => {
  return useQuery({
    queryKey: ['customer', href],
    queryFn: () => getCustomerByHref({ href }),
    enabled: !!href,
  })
}
```

## Use select for Transformations

### Don't Transform in queryFn

```tsx
// ❌ Bad: Transformation in queryFn (all components get this shape)
export const useGetProductsQuery = () => {
  return useQuery({
    queryKey: ['products'],
    queryFn: async () => {
      const data = await fetchProducts()
      return data.map((product) => ({
        ...product,
        displayName: `${product.name} - ${product.sku}`,
        isAvailable: product.stock > 0,
      }))
    },
  })
}

// ✅ Good: Keep raw data, use select for transformations
export const useGetProductsQuery = () => {
  return useQuery({
    queryKey: ['products'],
    queryFn: () => fetchProducts(), // Raw data
  })
}

// Component A: Needs display names
const ProductList = () => {
  const { data: displayProducts } = useQuery({
    ...useGetProductsQueryOptions(),
    select: (data) =>
      data.map((p) => ({
        ...p,
        displayName: `${p.name} - ${p.sku}`,
      })),
  })
}

// Component B: Needs different shape
const CheckoutSummary = () => {
  const { data: checkoutProducts } = useQuery({
    ...useGetProductsQueryOptions(),
    select: (data) =>
      data.map((p) => ({
        id: p.id,
        name: p.name,
        total: p.price * p.quantity,
      })),
  })
}
```

## Configure staleTime and gcTime

### Don't Use Defaults

```tsx
// ❌ Bad: Default staleTime is 0 - data is ALWAYS stale
export const useGetUserQuery = () => {
  return useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
    // staleTime defaults to 0 - EVERY mount triggers refetch!
  })
}

// ✅ Good: Configure based on data characteristics
// User data - changes infrequently
export const useGetUserQuery = () => {
  return useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 10 * 60 * 1000, // 10 minutes in cache
  })
}

// Product inventory - changes frequently
export const useGetProductAvailabilityQuery = (sku: string) => {
  return useQuery({
    queryKey: ['product-availability', sku],
    queryFn: () => fetchAvailability(sku),
    staleTime: 30 * 1000, // 30 seconds
    refetchInterval: 60 * 1000, // Auto-refetch every minute
  })
}

// Static content - rarely changes
export const useGetTermsQuery = () => {
  return useQuery({
    queryKey: ['terms-and-conditions'],
    queryFn: fetchTerms,
    staleTime: Infinity, // Never goes stale
    gcTime: Infinity, // Keep forever
  })
}

// Real-time data - needs to be fresh
export const useGetOrderStatusQuery = (orderId: string) => {
  return useQuery({
    queryKey: ['order-status', orderId],
    queryFn: () => fetchOrderStatus(orderId),
    staleTime: 0, // Always stale
    refetchInterval: 5 * 1000, // Poll every 5 seconds
    refetchOnWindowFocus: true,
  })
}
```

## Keep API Data in React Query, Not Global State

```tsx
// ❌ Bad: Syncing React Query data to Zustand
const useGetShippingAddressesQuery = () => {
  const { setOrder } = useGlobalStore()

  const { data: shipToOptions } = useQuery({
    queryKey: ['shipping-addresses'],
    queryFn: fetchAddresses,
  })

  // Anti-pattern: Multiple sources of truth
  useEffect(() => {
    if (shipToOptions) {
      const defaultAddress = shipToOptions.find((a) => a.isDefault)
      setOrder({ shipping: defaultAddress }) // Syncing to store!
    }
  }, [shipToOptions])
}

// ✅ Good: Keep API data ONLY in React Query
const useGetShippingAddressesQuery = () => {
  const { data: shipToOptions, isLoading } = useQuery({
    queryKey: ['shipping-addresses'],
    queryFn: fetchAddresses,
  })

  // Derive default address during render
  const defaultAddress = shipToOptions?.find((a) => a.isDefault) ?? null

  return { shipToOptions, defaultAddress, isLoading }
}

// Use directly in component
const CheckoutShipping = () => {
  const { shipToOptions, defaultAddress } = useGetShippingAddressesQuery()
  return <AddressSelector addresses={shipToOptions} default={defaultAddress} />
}
```

## Keep Query Functions Pure

```tsx
// ❌ Bad: Business logic and side effects in queryFn
export const useGetShippingAddressesQuery = () => {
  const { setOrder, setCheckout } = useGlobalStore()

  return useQuery({
    queryKey: ['shipping-addresses'],
    queryFn: async () => {
      const addresses = await fetchAddresses()

      // Side effects in queryFn!
      const defaultAddress = addresses.find((a) => a.isDefault)
      if (defaultAddress) {
        setOrder({ shipping: defaultAddress })
        setCheckout({ isAddingOrEditingShippingAddress: 'address_added' })
      }

      return addresses.filter((addr) => addr.address1 && addr.city)
    },
  })
}

// ✅ Good: Pure query function
export const useGetShippingAddressesQuery = () => {
  return useQuery({
    queryKey: ['shipping-addresses'],
    queryFn: () => fetchAddresses(), // Just fetch, no logic
    staleTime: 5 * 60 * 1000,
  })
}

// Use data directly in component
const CheckoutShipping = () => {
  const { data: addresses, isLoading } = useGetShippingAddressesQuery()
  const defaultAddress = addresses?.find((a) => a.isDefault)

  if (isLoading) return <Spinner />
  if (!defaultAddress) return <AddAddressForm />

  return <AddressDisplay address={defaultAddress} />
}
```

## Don't Sync Loading State to Global Store

```tsx
// ❌ Bad: Syncing React Query loading state to Zustand
useEffect(() => {
  setLoaders({
    isLoadingSavedAddresses: isLoadingShipToOptions,
  })
}, [isLoadingShipToOptions])

// ✅ Good: Use loading state directly from query
const CheckoutPage = () => {
  const { shipToOptions, isLoading } = useGetShippingAddressesQuery()

  if (isLoading) return <Spinner />
  return <AddressSelector addresses={shipToOptions} />
}
```

## Use Query Invalidation, Not Manual Refetch

```tsx
// ❌ Bad: Manual refetch
const { refetch: refetchUser } = useGetUserQuery()
const { refetch: refetchOrders } = useGetOrdersQuery()

const updateProfile = async (data) => {
  await updateUserProfile(data)
  refetchUser() // Manual
  refetchOrders() // Have to remember all affected queries
}

// ✅ Good: Use invalidation
const queryClient = useQueryClient()

const updateProfile = async (data) => {
  await updateUserProfile(data)
  queryClient.invalidateQueries({ queryKey: ['user'] })
  queryClient.invalidateQueries({ queryKey: ['orders'] })
}

// ✅ Better: Use mutation with onSuccess
const { mutate: updateProfile } = useMutation({
  mutationFn: updateUserProfile,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['user'] })
    queryClient.invalidateQueries({ queryKey: ['orders'] })
  },
})
```

## Use Hierarchical Query Keys

```tsx
// ✅ Good: Hierarchical key structure
// Format: ['entity', 'detail', ...filters]

// All users
;['users'][
  // Specific user
  ('users', userId)
][
  // User's orders
  ('users', userId, 'orders')
][
  // Specific order
  ('users', userId, 'orders', orderId)
][
  // All products
  'products'
][
  // Filtered products
  ('products', { category: 'supplements' })
][
  // Specific product
  ('products', productId)
][
  // Product reviews
  ('products', productId, 'reviews')
]

// Invalidation becomes easy:
queryClient.invalidateQueries({ queryKey: ['users'] }) // All user queries
queryClient.invalidateQueries({ queryKey: ['users', userId] }) // Specific user and nested
queryClient.invalidateQueries({ queryKey: ['products', productId] }) // Product and reviews
```

## What Should Be in Global State vs React Query

```tsx
// ✅ React Query: Server state (API data)
// - Customer data
// - Products
// - Orders
// - Shipping addresses
// - Any data from external APIs

// ✅ Global Store (Zustand): Client state
type AppType = {
  // UI state
  isCartOpen: boolean
  isLoginPopoverOpen: boolean
  isSearchBarActive: boolean

  // Client-side preferences
  selectedLocale: string
  theme: 'light' | 'dark'

  // Authentication tokens (needed for requests)
  token: string | null

  // User preferences
  keepSignedIn: boolean
}
```

## Use .mutate() with Callbacks Instead of .mutateAsync()

```tsx
// ❌ Bad: mutateAsync with manual state management
const [isLoading, setIsLoading] = useState(false)
const [formError, setFormError] = useState<string | null>(null)

const handleSubmit = async (data: FormData) => {
  setIsLoading(true)
  setFormError(null)
  try {
    await mutation.mutateAsync(data)
  } catch (error) {
    setFormError(error.message)
  } finally {
    setIsLoading(false)
  }
}

// ✅ Good: .mutate() with callbacks, derive state from mutation
const mutation = useMutation({ mutationFn: createOrder })
const formError = getErrorMessage(mutation.error)

const handleSubmit = (data: FormData) => {
  mutation.mutate(data, {
    onSuccess: (result) => {
      closeModal()
      navigateTo(result.id)
    },
  })
}
```

Why: React Query already tracks `isPending`, `error`, and `isSuccess`. Using `.mutateAsync()` with try/catch duplicates state that React Query manages for free.

## Separate Hook-Level and Per-Call Mutation Callbacks

Use hook-level callbacks for shared concerns (cache invalidation), per-call callbacks for context-specific behavior (UI updates):

```tsx
// Hook level — runs for ALL calls
export const useCreateOrderMutation = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createOrder,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['orders'] })
    },
  })
}

// Per-call level — runs for THIS call only
mutation.mutate(data, {
  onSuccess: (result) => {
    closeModal()
    navigateTo(result.id)
  },
})
```

Why: Both callbacks run on success. Hook-level handles global concerns (cache), per-call handles local concerns (UI). This keeps the hook reusable across different call sites.

## Keep Mutation Hooks as Thin API Wrappers

Mutation hooks should only call APIs. Move business logic to helper functions or components:

```tsx
// ✅ Good: Hook just calls the API
export const useVerifyAddressMutation = () => {
  const config = useConfig()

  return useMutation({
    mutationFn: async (params: VerifyAddressParams) => {
      return verifyAddress(config, params)
    },
  })
}

// ❌ Bad: Business logic inside mutation
export const useVerifyAddressMutation = () => {
  return useMutation({
    mutationFn: async (params) => {
      const response = await fetchAddress(params)
      // This logic doesn't belong here
      const hasDifferences = compareAddresses(params, response)
      return { status: hasDifferences ? 'corrected' : 'valid', suggested: response }
    },
  })
}
```

Why: Thin hooks are reusable across different use cases. Business logic is testable without mocking hooks.

## Derive State from Mutations Instead of useState

Don't create separate state for things React Query already tracks:

```tsx
// ✅ Good: Derive from mutation state
const mutation = useMutation({ mutationFn: saveData })
const isLoading = mutation.isPending
const formError = getErrorMessage(mutation.error)

// ❌ Bad: Duplicate state that mirrors mutation
const [isLoading, setIsLoading] = useState(false)
const [formError, setFormError] = useState<string | null>(null)
```

Why: Duplicate state creates sync bugs and extra code. React Query already manages `isPending`, `error`, and `isSuccess`.

## Avoid Wrapping Mutations in useCallback

The `mutation.mutate` function reference is stable. Don't wrap it:

```tsx
// ✅ Good: Call mutation directly
const handleSave = () => {
  mutation.mutate(buildArgs(formData))
}

// ❌ Bad: Unnecessary wrapper
const saveAddress = useCallback(async (data: FormData) => {
  await mutation.mutateAsync(buildArgs(data))
}, [mutation])
```

Why: `useCallback` adds indirection without benefit. If you need to transform data, do it inline or use a simple function.

## Type Mutation Errors Explicitly

Specify the error type in the mutation generic so callbacks and error handling get proper types:

```tsx
// ✅ Good: Explicit error type
useMutation<CreateOrderResponse, HydraError, CreateOrderArgs>({
  mutationFn: createOrder,
  onError: (error) => {
    // error is typed as HydraError
    logger.error('[Checkout] Order failed', { error })
  },
})

// ❌ Bad: Error defaults to Error
useMutation({
  mutationFn: createOrder,
})
```

Why: Explicit error types enable proper error handling and TypeScript inference in callbacks.

## Decision Trees

### .mutate() vs .mutateAsync()

```
Do I need the result immediately in this function?
├─ YES → Do I have complex sequential logic?
│  ├─ YES → mutateAsync with proper error handling
│  └─ NO → mutate with onSuccess callback (preferred)
└─ NO → mutate() is always fine
```

### Where Should Mutation Logic Live?

```
Is it API call mechanics?
├─ YES → In the mutation hook
└─ NO → Is it data transformation for the API?
   ├─ YES → Helper function, called when invoking mutation
   └─ NO → Is it response transformation for UI?
      ├─ YES → Helper function, called in onSuccess
      └─ NO → Component logic
```

### Should I Use useState for This?

```
Is it already tracked by a hook/mutation?
├─ YES → Derive from existing state
│        (mutation.error, mutation.isPending, form.formState)
└─ NO → Does it need to persist across renders?
   ├─ YES → Is it form field state?
   │  ├─ YES → Use React Hook Form
   │  └─ NO → useState is appropriate
   └─ NO → Local variable is fine
```
