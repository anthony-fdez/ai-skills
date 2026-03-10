---
name: writing-nextjs
description: Enforces Next.js Pages Router patterns for pages, data fetching, static generation, and page-level concerns. Use when creating pages, implementing getStaticProps/getServerSideProps, writing middleware, optimizing with ISR, adding SEO meta tags, creating error pages, or configuring internationalization. Note: for API route handlers (src/app/api/), use the designing-apis skill instead — this skill covers page-level patterns only.
---

# Writing Next.js (Pages Router)

Patterns for Next.js Pages Router including page structure, data fetching, and page-level concerns.

## Contents

- [Follow Standard Page Structure](#page-structure)
- [Use Standard Page Pattern with Head and SEO](#standard-page-pattern)
- [Prefer Static Generation with ISR](#data-fetching)
- [Use Server-Side Rendering Only When Needed](#server-side-rendering-when-needed)
- [Add SEO and Meta Tags](#seo--meta-tags)
- [Create Custom Error Pages](#error-pages)
- [Lazy Load Heavy Components](#lazy-load-heavy-components)
- [Optimize Images with next/image](#optimize-images)
- [Use Translations for All User-Facing Text](#internationalization)

---

## Page Structure

```
pages/
├── _app.tsx        # App wrapper
├── _document.tsx   # HTML document
├── index.tsx       # Home page
├── 404.tsx         # Custom 404 page
├── api/            # API routes
└── [locale]/       # Dynamic locale routes
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

Why: Static generation serves pre-built HTML from the CDN — it's the fastest option. ISR gives you the speed of static with the freshness of dynamic.

### Dynamic Routes with getStaticPaths

```tsx
// [locale]/index.tsx
export const getStaticPaths: GetStaticPaths = async () => {
  return {
    paths: locales.map((locale) => ({
      params: { locale: locale.code },
    })),
    fallback: 'blocking', // Generate missing pages on-demand
  }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const locale = params?.locale as string
  const data = await fetchLocaleData(locale)

  return {
    props: { data, locale },
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

For API route handlers, see the **designing-apis** skill which covers the standard patterns for `src/app/api/` route handlers with Zod validation and consistent error responses.

This skill focuses on page-level concerns only (data fetching, SEO, middleware, etc.).

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

Why: Heavy components (charts, editors, maps) block initial page load. Dynamic imports with ssr: false keep them out of the server bundle entirely.

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
// Middleware sets locale based on [locale] param
// Access via useTranslate() hook
const translate = useTranslate()
const { locale } = translate
```

## Analytics Integration

```tsx
// Pages automatically track page views
// Add custom events:
import { track } from '@/lib/hooks/analytics'

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
