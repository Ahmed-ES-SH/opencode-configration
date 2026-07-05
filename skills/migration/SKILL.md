---
name: migration
description: Handles Next.js version upgrades and Pages Router to App Router migration. Use when upgrading Next.js, migrating from Pages Router to App Router, replacing deprecated APIs, using codemods, or handling breaking changes.
---

# Migration Skill

## Trigger

Use when the request mentions: **upgrading Next.js, migrating from Pages Router to App Router, replacing deprecated APIs, using codemods, incremental adoption, or version-specific breaking changes**.

---

## Scope

- Pages Router → App Router migration strategy
- Next.js version upgrade guides (13 → 14 → 15)
- Deprecated API replacements
- Codemod usage
- Incremental migration patterns
- Breaking changes in Next.js 15 (async params, cookies, headers)

---

## Next.js 15 Breaking Changes

### 1. Async `params` and `searchParams`

```typescript
// ❌ Next.js 14 (sync)
export default function Page({ params }: { params: { id: string } }) {
  const { id } = params
}

// ✅ Next.js 15 (async)
export default async function Page({
  params,
  searchParams,
}: {
  params: Promise<{ id: string }>
  searchParams: Promise<{ page: string }>
}) {
  const { id } = await params
  const { page } = await searchParams
}
```

### 2. Async `cookies()` and `headers()`

```typescript
// ❌ Next.js 14 (sync)
import { cookies, headers } from 'next/headers'
const token = cookies().get('token')
const auth = headers().get('authorization')

// ✅ Next.js 15 (async)
const cookieStore = await cookies()
const token = cookieStore.get('token')
const headerList = await headers()
const auth = headerList.get('authorization')
```

### 3. fetch Cache Default Changed

```typescript
// In Next.js 15, fetch() defaults to no-store (was force-cache in 14)
// If you relied on default caching, add explicit options

// ✅ Be explicit — don't rely on defaults
fetch(url, { cache: 'force-cache' })            // Static
fetch(url, { next: { revalidate: 60 } })        // ISR
fetch(url, { cache: 'no-store' })               // Dynamic
```

### 4. React 19 in Next.js 15

```typescript
// forwardRef no longer needed — ref is a regular prop
// ❌ Old
const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => (
  <input ref={ref} {...props} />
))

// ✅ React 19
function Input({ ref, ...props }: InputProps & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />
}

// use() hook replaces some patterns
import { use } from 'react'
function Component({ promise }: { promise: Promise<Data> }) {
  const data = use(promise) // Suspense-aware
}
```

---

## Pages Router → App Router Migration

### Strategy: Incremental Migration

Next.js supports both routers simultaneously. Migrate feature-by-feature.

```
Phase 1: Set up App Router alongside Pages Router
Phase 2: Migrate shared layouts first (app/layout.tsx)
Phase 3: Migrate static/simple pages
Phase 4: Migrate data-fetching pages (replace getServerSideProps/getStaticProps)
Phase 5: Migrate API routes → Route Handlers
Phase 6: Remove pages/ directory
```

### getServerSideProps → Server Component

```typescript
// ❌ Pages Router
// pages/orders/index.tsx
export async function getServerSideProps(context: GetServerSidePropsContext) {
  const session = await getSession(context)
  if (!session) return { redirect: { destination: '/login', permanent: false } }

  const orders = await fetchOrders(session.accessToken)
  return { props: { orders } }
}

export default function OrdersPage({ orders }: { orders: Order[] }) {
  return <OrdersTable orders={orders} />
}

// ✅ App Router
// app/orders/page.tsx
export default async function OrdersPage() {
  const session = await getServerSession(authOptions)
  if (!session) redirect('/login')

  const orders = await apiFetch<Order[]>('/api/orders', {
    headers: { Authorization: `Bearer ${session.accessToken}` },
  })

  return <OrdersTable orders={orders} />
}
```

### getStaticProps + getStaticPaths → generateStaticParams

```typescript
// ❌ Pages Router
// pages/products/[id].tsx
export async function getStaticPaths() {
  const products = await getProducts()
  return {
    paths: products.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking',
  }
}

export async function getStaticProps({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)
  return { props: { product }, revalidate: 60 }
}

// ✅ App Router
// app/products/[id]/page.tsx
export async function generateStaticParams() {
  const products = await apiFetch<Product[]>('/api/products')
  return products.map(p => ({ id: p.id }))
}

export const revalidate = 60

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const product = await apiFetch<Product>(`/api/products/${id}`)
  return <ProductDetail product={product} />
}
```

### API Routes → Route Handlers

```typescript
// ❌ Pages Router
// pages/api/orders.ts
import type { NextApiRequest, NextApiResponse } from 'next'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    const orders = await getOrders()
    res.status(200).json(orders)
  } else if (req.method === 'POST') {
    const order = await createOrder(req.body)
    res.status(201).json(order)
  }
}

// ✅ App Router
// app/api/orders/route.ts
export async function GET(req: NextRequest) {
  const orders = await getOrders()
  return NextResponse.json(orders)
}

export async function POST(req: NextRequest) {
  const body = await req.json()
  const order = await createOrder(body)
  return NextResponse.json(order, { status: 201 })
}
```

### _app.tsx and _document.tsx → app/layout.tsx

```typescript
// ❌ Pages Router
// pages/_app.tsx
export default function App({ Component, pageProps: { session, ...pageProps } }: AppProps) {
  return (
    <SessionProvider session={session}>
      <ThemeProvider>
        <Component {...pageProps} />
      </ThemeProvider>
    </SessionProvider>
  )
}

// ✅ App Router
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <SessionProvider>        {/* Must be "use client" wrapper */}
          <ThemeProvider>
            {children}
          </ThemeProvider>
        </SessionProvider>
      </body>
    </html>
  )
}
```

---

## Running Codemods

```bash
# Upgrade to Next.js 15 with codemod
npx @next/codemod@latest upgrade

# Specific codemods
npx @next/codemod next-async-request-api .   # Fix async params/cookies/headers
npx @next/codemod next-image-to-legacy-image . # Image API migration
npx @next/codemod built-in-next-font .         # next/font migration

# Run all available codemods for a version
npx @next/codemod@canary --list
```

---

## Deprecated APIs → Replacements

| Deprecated (Pages Router)         | Replacement (App Router)                     |
|-----------------------------------|----------------------------------------------|
| `getServerSideProps`              | `async` Server Component + `cache: 'no-store'` |
| `getStaticProps`                  | `async` Server Component + `revalidate`      |
| `getStaticPaths`                  | `generateStaticParams()`                    |
| `next/router` (useRouter)         | `next/navigation` (useRouter + usePathname) |
| `pages/api/*.ts`                  | `app/api/*/route.ts`                        |
| `_app.tsx`                        | `app/layout.tsx`                            |
| `_document.tsx`                   | `app/layout.tsx` html/body                  |
| `next/head`                       | `metadata` export or `generateMetadata()`  |
| `getSession()` (next-auth/client) | `getServerSession(authOptions)` (server)    |

---

## Notes

- **Never migrate everything at once** — incremental is safer and shippable
- **Test each migrated route before moving to the next**
- **Keep the `pages/` directory until fully migrated** — both routers work together
- **Check next-auth compatibility** — v5 (Auth.js) has a different API, plan the upgrade separately
