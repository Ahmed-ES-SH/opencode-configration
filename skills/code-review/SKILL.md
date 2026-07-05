---
name: code-review
description: Audits code for security vulnerabilities, performance anti-patterns, TypeScript safety, and production readiness. Use when reviewing PRs, auditing code, assessing quality, or evaluating architectural decisions.
---

# CodeReview Skill

## Trigger

Use when the request asks to: **review code, audit a PR, find bugs or issues, assess code quality, check if implementation is production-ready, or evaluate architectural decisions**.

---

## Scope

- Security vulnerabilities
- Performance anti-patterns
- TypeScript type safety
- Auth/session misuse
- Fetch and async error handling
- Component design issues
- Missing validation
- Accessibility gaps
- Test coverage assessment
- Architecture violations

---

## Review Checklist

### Security
```
□ No secrets or API keys in client-side code (no NEXT_PUBLIC_ on secrets)
□ All Route Handlers check authentication before processing
□ Authorization checked (not just authentication — does THIS user own THIS resource?)
□ All user inputs validated with Zod before use
□ No dangerouslySetInnerHTML with unscanitized content
□ Env vars accessed server-side only where sensitive
□ Cookie options: httpOnly, sameSite, secure (prod)
```

### TypeScript
```
□ No `any` types — especially on API responses
□ Props interfaces defined for all components
□ Return types on async functions
□ Zod schemas used for runtime validation (types alone aren't enough)
□ Next-auth types augmented if session data accessed
```

### Performance
```
□ "use client" is justified — not applied unnecessarily
□ fetch() uses caching/revalidation strategy (not always no-store)
□ Heavy components (charts, editors) are dynamically imported
□ next/image used for all images with dimensions set
□ LCP image has priority prop
□ No useEffect for data that could be server-fetched
```

### Error Handling
```
□ fetch() responses checked with res.ok before .json()
□ Server Actions return structured error objects, not throw
□ error.tsx exists on routes that fetch data
□ loading.tsx exists on routes with async data
□ Empty states handled for all lists/tables
```

### Accessibility
```
□ Interactive elements are keyboard accessible
□ All images have alt text (empty for decorative)
□ Form inputs have associated labels (htmlFor + id)
□ Error messages associated with inputs (aria-describedby)
□ No color-only meaning without text/icon backup
```

---

## Common Issues Found in Reviews

### Issue 1: Unprotected Route Handler

```typescript
// ❌ No auth check
export async function DELETE(req: NextRequest, { params }: { params: { id: string } }) {
  await db.orders.delete(params.id) // Anyone can delete any order!
}

// ✅ Always check auth + authorization
export async function DELETE(req: NextRequest, { params }: { params: { id: string } }) {
  const session = await getServerSession(authOptions)
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const order = await getOrder(params.id)
  if (order.userId !== session.user.id && session.user.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  await db.orders.delete(params.id)
  return NextResponse.json({ success: true })
}
```

### Issue 2: Unhandled fetch Errors

```typescript
// ❌ No error handling
export default async function OrderPage({ params }: { params: { id: string } }) {
  const data = await fetch(`${process.env.API_URL}/api/orders/${params.id}`).then(r => r.json())
  return <OrderDetail order={data} />
}

// ✅ Handle errors explicitly
export default async function OrderPage({ params }: { params: { id: string } }) {
  const { id } = await params
  const res = await fetch(`${process.env.API_URL}/api/orders/${id}`, { cache: 'no-store' })

  if (res.status === 404) notFound()
  if (!res.ok) throw new Error('Failed to load order') // caught by error.tsx

  const order: Order = await res.json()
  return <OrderDetail order={order} />
}
```

### Issue 3: Client Component Overuse

```typescript
// ❌ Entire page is "use client" just for one interactive button
'use client'
export default function OrdersPage() {
  // Fetches data client-side unnecessarily
  const [orders, setOrders] = useState([])
  useEffect(() => { fetch('/api/orders').then(...) }, [])
  return (
    <div>
      <h1>Orders</h1>
      <table>...</table>
      <button onClick={() => {}}>New Order</button>
    </div>
  )
}

// ✅ Server Component with isolated client boundary
// app/orders/page.tsx (Server Component)
export default async function OrdersPage() {
  const orders = await apiFetch<Order[]>('/api/orders', { next: { revalidate: 60 } })
  return (
    <div>
      <h1>Orders</h1>
      <OrdersTable orders={orders} />
      <NewOrderButton />  {/* Small "use client" component */}
    </div>
  )
}
```

### Issue 4: Missing Input Validation

```typescript
// ❌ Using form data directly without validation
export async function updateUser(formData: FormData) {
  const name = formData.get('name') as string
  const role = formData.get('role') as string // User can set any role!
  await db.users.update({ name, role })
}

// ✅ Validate and restrict what can be changed
const UpdateUserSchema = z.object({
  name: z.string().min(1).max(100),
  // role is NOT in this schema — users can't change their own role
})

export async function updateUser(formData: FormData) {
  const session = await getServerSession(authOptions)
  if (!session) throw new Error('Unauthorized')

  const parsed = UpdateUserSchema.safeParse({ name: formData.get('name') })
  if (!parsed.success) return { error: parsed.error.flatten().fieldErrors }

  await apiFetch(`/api/users/${session.user.id}`, {
    method: 'PATCH',
    body: JSON.stringify(parsed.data),
  })
}
```

### Issue 5: `any` on API Responses

```typescript
// ❌ Untyped response
const data: any = await apiFetch('/api/orders')
data.items.map((i: any) => ...) // No type safety, runtime errors waiting

// ✅ Typed response
interface PaginatedOrders {
  data: Order[]
  meta: { total: number; per_page: number; current_page: number }
}

const data = await apiFetch<PaginatedOrders>('/api/orders')
data.data.map(order => ...) // Fully typed
```

---

## Production-Readiness Checklist

```
□ Error boundaries (error.tsx) on all data-fetching routes
□ Loading states (loading.tsx or Suspense) on all async routes
□ 404 handling (notFound()) for resource-not-found cases
□ not-found.tsx at app root
□ Security headers configured in next.config.ts
□ Env vars documented in .env.example (no real values)
□ TypeScript strict mode with no errors
□ No console.log statements in production code
□ Rate limiting on auth and public API routes
□ robots.txt and sitemap.xml for public apps
□ Bundle analyzed — no unexpected large dependencies
```

---

## Notes

- **Authorization ≠ Authentication** — always verify the user owns the resource, not just that they're logged in
- **Fail closed** — if auth check throws, deny access; never default to allowing
- **Review the happy path AND error paths** — most bugs live in error handling
- **Check the `.env.example`** — ensure no real secrets are committed
