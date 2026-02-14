---
name: writing-nextjs
description: Enforces Next.js Pages Router patterns for pages, data fetching, and API routes. Use when creating pages, implementing data fetching, or writing API routes.
---

# Writing Next.js (Pages Router)

Patterns for Next.js Pages Router including page structure, data fetching, and API routes.

## Page Structure

```
pages/
├── _app.tsx        # App wrapper
├── _document.tsx   # HTML document
├── index.tsx       # Home page
├── 404.tsx         # Custom 404 page
├── api/            # API routes
├── [alpha3]/       # Dynamic country routes
├── reset/          # Application reset
└── unsupported/    # Unsupported market
```

## Standard Page Pattern

```tsx
import type { GetStaticProps } from 'next'
import Head from 'next/head'

type PageProps = {
  data: DataType
}

const PageName = ({ data }: PageProps) => {
  const translate = useTranslate()

  return (
    <>
      <Head>
        <title>{translate('page.title')}</title>
        <meta name="description" content={translate('page.description')} />
        <meta property="og:title" content={translate('page.title')} />
        <link rel="canonical" href={canonicalUrl} />
      </Head>

      <main>{/* Page content */}</main>
    </>
  )
}

export const getStaticProps: GetStaticProps<PageProps> = async () => {
  const data = await fetchData()

  return {
    props: { data },
    revalidate: 3600, // ISR: regenerate every hour
  }
}

export default PageName
```

## Data Fetching

### Static Generation (Preferred)

```tsx
// ✅ Good: Use getStaticProps for data that doesn't change per-request
export const getStaticProps: GetStaticProps = async () => {
  const data = await fetchData()

  return {
    props: { data },
    revalidate: 3600, // ISR in seconds
  }
}
```

### Dynamic Routes with getStaticPaths

```tsx
// [alpha3]/index.tsx
export const getStaticPaths: GetStaticPaths = async () => {
  return {
    paths: countries.map((country) => ({
      params: { alpha3: country.code },
    })),
    fallback: 'blocking', // Generate missing pages on-demand
  }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const country = params?.alpha3 as string
  const data = await fetchCountryData(country)

  return {
    props: { data, country },
    revalidate: 3600,
  }
}
```

### Server-Side Rendering (When Needed)

```tsx
// ✅ Use only when data must be fresh on every request
export const getServerSideProps: GetServerSideProps = async (context) => {
  const { query, req, res } = context

  // Access cookies, headers, etc.
  const token = req.cookies.authToken

  const data = await fetchUserSpecificData(token)

  return {
    props: { data },
  }
}
```

## API Routes

### Standard Pattern

```tsx
// api/endpoint.api.ts (note .api.ts extension)
import type { NextApiRequest, NextApiResponse } from 'next'

type ResponseData = {
  success: boolean
  data?: DataType
  error?: string
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<ResponseData>,
) {
  // 1. Check HTTP method
  if (req.method !== 'POST') {
    return res.status(405).json({ success: false, error: 'Method not allowed' })
  }

  // 2. Validate request body
  const { email, password } = req.body
  if (!email || !password) {
    return res
      .status(400)
      .json({ success: false, error: 'Missing required fields' })
  }

  try {
    // 3. Handle request
    const data = await processRequest(req.body)

    // 4. Return success
    res.status(200).json({ success: true, data })
  } catch (error) {
    // 5. Handle errors
    res.status(500).json({ success: false, error: 'Internal server error' })
  }
}
```

## SEO & Meta Tags

```tsx
import Head from 'next/head'
;<Head>
  {/* Essential */}
  <title>{translate('page.title')}</title>
  <meta name="description" content={description} />

  {/* Open Graph */}
  <meta property="og:title" content={title} />
  <meta property="og:description" content={description} />
  <meta property="og:image" content={imageUrl} />
  <meta property="og:type" content="website" />

  {/* Twitter */}
  <meta name="twitter:card" content="summary_large_image" />

  {/* Canonical */}
  <link rel="canonical" href={canonicalUrl} />
</Head>
```

## Error Pages

### Custom 404

```tsx
// 404.tsx
const Custom404 = () => {
  const translate = useTranslate()

  return (
    <div className="flex min-h-screen flex-col items-center justify-center">
      <h1 className="text-4xl font-bold">404</h1>
      <p className="mt-2">{translate('error.page_not_found')}</p>
      <Link href="/" className="mt-4 text-primary underline">
        {translate('error.go_home')}
      </Link>
    </div>
  )
}

export default Custom404
```

### Error Boundary in \_app.tsx

```tsx
import { ErrorBoundary } from 'react-error-boundary'

const MyApp = ({ Component, pageProps }: AppProps) => {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Component {...pageProps} />
    </ErrorBoundary>
  )
}
```

## Middleware

```tsx
// middleware.tsx
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // Country detection
  const country = request.geo?.country || 'US'

  // Authentication check
  const token = request.cookies.get('authToken')

  // Redirect if needed
  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

## Performance Optimization

### Use Static Generation When Possible

```tsx
// ✅ Good: Static for product pages (ISR for updates)
export const getStaticProps = async () => {
  return { props: { data }, revalidate: 3600 }
}

// ❌ Avoid: SSR when static would work
export const getServerSideProps = async () => {
  return { props: { data } } // Slower, no caching
}
```

### Lazy Load Heavy Components

```tsx
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <Skeleton className="h-64" />,
  ssr: false, // Client-only component
})
```

### Optimize Images

```tsx
import Image from 'next/image'
;<Image
  src={product.image}
  alt={product.name}
  width={400}
  height={300}
  priority={isAboveFold} // Preload important images
  placeholder="blur"
  blurDataURL={product.blurHash}
/>
```

## Internationalization

### Using Translations

```tsx
const ProductPage = () => {
  const translate = useTranslate()

  return (
    <div>
      <h1>{translate('product.title')}</h1>
      <p>{translate('product.description')}</p>
    </div>
  )
}
```

### Locale from URL

```tsx
// Middleware sets locale based on [alpha3] param
// Access via useTranslate() hook
const translate = useTranslate()
const { locale, alpha2 } = translate
```

## Analytics Integration

```tsx
// Pages automatically track page views
// Add custom events:
import { track } from '@/lib/hooks/mixpanel'

const handlePurchase = () => {
  track('Purchase Completed', {
    orderId: order.id,
    total: order.total,
  })
}
```

## Checklist

- [ ] Page uses appropriate data fetching method (getStaticProps preferred)
- [ ] ISR configured with revalidate for dynamic content
- [ ] Head component includes title, description, and og tags
- [ ] Canonical URL set for SEO
- [ ] API routes check HTTP method
- [ ] API routes return consistent response format
- [ ] Heavy components lazy loaded with dynamic()
- [ ] Images use next/image with proper dimensions
- [ ] Translations used for all user-facing text
- [ ] Error boundary wraps page content
