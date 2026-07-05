---
name: performance
description: Optimizes Core Web Vitals, bundle size, rendering, and runtime efficiency for Next.js applications. Use when discussing performance, Core Web Vitals, bundle size, caching, streaming, image/font optimization, code splitting, or memoization.
---

# Performance Skill

## Trigger

Use when the request mentions: **performance, Core Web Vitals, bundle size, render optimization, caching, prefetching, streaming, image optimization, font loading, lazy loading, memoization, or runtime efficiency**.

---

## Scope

- Core Web Vitals (LCP, FID/INP, CLS)
- Server Component render strategy
- Bundle analysis and tree-shaking
- Image and font optimization
- Caching strategies (fetch cache, React cache, unstable_cache)
- Streaming and progressive rendering
- Code splitting and lazy loading
- React memoization (`memo`, `useMemo`, `useCallback`)
- Database query optimization (N+1 awareness)

---

## Core Web Vitals Targets

| Metric | Good    | Needs Work | Poor    |
|--------|---------|------------|---------|
| LCP    | < 2.5s  | < 4.0s     | > 4.0s  |
| INP    | < 200ms | < 500ms    | > 500ms |
| CLS    | < 0.1   | < 0.25     | > 0.25  |

---

## Server Components First (Biggest Win)

```typescript
// ✅ Default: Server Component — zero JS sent to browser
// Fetches on server, renders HTML, no hydration overhead
async function ProductList() {
  const products = await apiFetch<Product[]>('/api/products', {
    next: { revalidate: 300 },
  })
  return <ul>{products.map(p => <ProductCard key={p.id} product={p} />)}</ul>
}

// Only add "use client" when you need interactivity
// Every "use client" boundary increases bundle size
```

---

## Image Optimization

```typescript
// Always use next/image — automatic WebP/AVIF, lazy load, size hints
import Image from 'next/image'

// ✅ Known dimensions
<Image
  src="/hero.jpg"
  alt="Hero banner"
  width={1200}
  height={630}
  priority              // LCP image: preload it
  className="rounded-lg"
/>

// ✅ Fill parent container (responsive)
<div className="relative aspect-video">
  <Image
    src={product.imageUrl}
    alt={product.name}
    fill
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    className="object-cover rounded"
  />
</div>

// ✅ Remote images — whitelist in next.config.ts
// next.config.ts
images: {
  remotePatterns: [
    { protocol: 'https', hostname: 'cdn.yourapp.com' },
  ],
}
```

---

## Font Optimization

```typescript
// app/layout.tsx — next/font handles: no layout shift, self-hosted, preloading
import { Inter, JetBrains_Mono } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-sans',
  display: 'swap',
})

const mono = JetBrains_Mono({
  subsets: ['latin'],
  variable: '--font-mono',
  display: 'swap',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${mono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  )
}
```

---

## Streaming & Suspense

```typescript
// Stream slow data without blocking fast data
// app/(dashboard)/page.tsx
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div className="grid grid-cols-3 gap-6">
      {/* Fast: renders immediately */}
      <StatsCards />

      {/* Slow: streamed in after fast content */}
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />   {/* This fetches slow data */}
      </Suspense>

      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders />   {/* Independent slow fetch */}
      </Suspense>
    </div>
  )
}
```

---

## Caching Strategy

```typescript
// 1. Static — cache forever, revalidate on deployment
fetch(url, { cache: 'force-cache' })

// 2. ISR — revalidate on timer
fetch(url, { next: { revalidate: 60 } })

// 3. Tag-based — revalidate on demand
fetch(url, { next: { tags: ['products', 'product-123'] } })
// Then call: revalidateTag('products') from a server action

// 4. No cache — always fresh
fetch(url, { cache: 'no-store' })

// 5. unstable_cache — cache expensive computations or non-fetch data
import { unstable_cache } from 'next/cache'

const getExpensiveData = unstable_cache(
  async (userId: string) => {
    // expensive DB or computation
    return computeUserMetrics(userId)
  },
  ['user-metrics'],
  { revalidate: 300, tags: ['user-metrics'] }
)
```

---

## Code Splitting & Lazy Loading

```typescript
import dynamic from 'next/dynamic'

// Lazy load heavy components (charts, editors, maps)
const RevenueChart = dynamic(() => import('@/components/features/RevenueChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // disable SSR for browser-only libs (e.g., chart.js)
})

const RichTextEditor = dynamic(() => import('@/components/ui/RichTextEditor'), {
  loading: () => <p>Loading editor...</p>,
  ssr: false,
})

// Load only on user interaction
const HeavyModal = dynamic(() => import('@/components/ui/HeavyModal'))
// Only loads the bundle when the modal is first opened
```

---

## React Memoization (Use Sparingly)

```typescript
// memo: prevent re-render when parent re-renders with same props
import { memo } from 'react'

export const OrderRow = memo(function OrderRow({ order }: { order: Order }) {
  return <tr>...</tr>
})
// ✅ Use for: list items, expensive renders
// ❌ Don't use on everything — profiling first

// useMemo: memoize expensive computations
const sortedOrders = useMemo(
  () => [...orders].sort((a, b) => b.total - a.total),
  [orders]
)
// ✅ Use for: sort/filter on large arrays, heavy transforms
// ❌ Don't use for: simple calculations, values used once

// useCallback: stable function reference
const handleDelete = useCallback(
  (id: string) => deleteOrder(id),
  [deleteOrder]
)
// ✅ Use when: passing callbacks to memoized children
// ❌ Don't over-use — check with React DevTools Profiler first
```

---

## Bundle Analysis

```bash
# Analyze bundle
npm install -D @next/bundle-analyzer

# next.config.ts
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})
module.exports = withBundleAnalyzer(nextConfig)

# Run
ANALYZE=true npm run build
```

**Common bundle wins:**
- Replace `lodash` with `lodash-es` (tree-shakeable)
- Use `date-fns` instead of `moment`
- Lazy load chart libraries
- Check for duplicate dependencies with `npm ls`

---

## Notes

- **Profile before optimizing** — use React DevTools Profiler + Chrome Performance tab
- **Avoid `useEffect` for data** — Server Components eliminate most fetch waterfalls
- **Avoid layout shift** — always set image dimensions, reserve space for async content
- **`priority` prop** on your LCP image — the single biggest LCP improvement
- **Prefetch critical routes** — Next.js Link prefetches by default in viewport
