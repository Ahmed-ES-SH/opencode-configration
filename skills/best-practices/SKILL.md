---
name: best-practices
description: Enforces senior-grade code quality and maintainability conventions for Next.js codebases. Use when discussing best practices, naming conventions, refactoring, anti-pattern detection, TypeScript strictness, or clean abstraction design.
---

# BestPractices Skill

## Trigger

Use when the request asks for: **best practices, code quality, maintainability, naming conventions, refactoring guidance, anti-pattern detection, folder conventions, senior-level review, or clean abstraction design**.

---

## Scope

This skill enforces **senior-grade code quality** across the entire Next.js codebase:

- File and folder naming conventions
- Component and hook design patterns
- TypeScript strictness and type safety
- Import organization and path aliases
- Anti-patterns to avoid
- Abstraction boundaries
- Clean async/await patterns
- Error handling conventions

---

## File & Folder Naming

```
app/                          # App Router — all routes live here
  (auth)/                     # Route group (no URL segment)
    login/page.tsx
    register/page.tsx
  (dashboard)/
    layout.tsx
    page.tsx
    orders/
      page.tsx
      [id]/
        page.tsx
        loading.tsx
        error.tsx
  api/
    orders/route.ts

components/
  ui/                         # Primitives: Button, Input, Badge, Modal
  layout/                     # Header, Sidebar, Footer
  features/                   # Feature-specific: OrdersTable, InvoiceCard
    orders/
      OrdersTable.tsx
      OrdersTable.test.tsx
      useOrders.ts
      types.ts

lib/
  auth.ts                     # NextAuth config
  api.ts                      # Central fetch wrapper
  utils.ts                    # cn(), formatDate(), etc.
  validations/                # Zod schemas

hooks/                        # Shared custom hooks
types/                        # Global type declarations
```

**Naming rules:**
- Files: `PascalCase` for components, `camelCase` for utils/hooks/lib
- Hooks: always `use` prefix — `useOrders.ts`, `useAuth.ts`
- Types: colocate with feature unless globally shared
- Test files: `ComponentName.test.tsx` next to the component

---

## TypeScript Strictness

```typescript
// tsconfig.json — always use strict mode
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}

// ❌ Never do this
const data: any = await fetchUser()
function process(input: any) { ... }

// ✅ Always type properly
interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'user' | 'viewer'
}

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`)
  if (!res.ok) throw new Error(`Failed to fetch user ${id}`)
  return res.json() as Promise<User>
}
```

---

## Central Fetch Wrapper

```typescript
// lib/api.ts
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'

interface FetchOptions extends RequestInit {
  params?: Record<string, string | number>
}

export async function apiFetch<T>(
  endpoint: string,
  options: FetchOptions = {}
): Promise<T> {
  const session = await getServerSession(authOptions)
  const { params, ...init } = options

  const url = new URL(`${process.env.API_URL}${endpoint}`)
  if (params) {
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, String(v)))
  }

  const res = await fetch(url.toString(), {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json',
      ...(session?.accessToken && { Authorization: `Bearer ${session.accessToken}` }),
      ...init.headers,
    },
  })

  if (!res.ok) {
    const error = await res.json().catch(() => ({ message: res.statusText }))
    throw new ApiError(res.status, error.message ?? 'Request failed')
  }

  return res.json() as Promise<T>
}

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message)
    this.name = 'ApiError'
  }
}
```

---

## Component Best Practices

```typescript
// ✅ Single responsibility — one component, one job
// ✅ Props interface always defined explicitly
// ✅ Default exports for pages, named exports for components

// components/features/orders/OrdersTable.tsx
interface OrdersTableProps {
  orders: Order[]
  onDelete?: (id: string) => void
  isLoading?: boolean
}

export function OrdersTable({ orders, onDelete, isLoading = false }: OrdersTableProps) {
  if (isLoading) return <OrdersTableSkeleton />
  if (!orders.length) return <EmptyState message="No orders found" />

  return (
    <table className="w-full text-sm">
      <thead>...</thead>
      <tbody>
        {orders.map(order => (
          <OrderRow key={order.id} order={order} onDelete={onDelete} />
        ))}
      </tbody>
    </table>
  )
}
```

---

## Custom Hook Pattern

```typescript
// hooks/useOrders.ts — encapsulate data + state, not UI
'use client'
import { useState, useCallback } from 'react'

export function useOrders() {
  const [orders, setOrders] = useState<Order[]>([])
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const fetchOrders = useCallback(async (page = 1) => {
    setIsLoading(true)
    setError(null)
    try {
      const data = await apiFetch<PaginatedResponse<Order>>('/api/orders', {
        params: { page },
      })
      setOrders(data.data)
    } catch (err) {
      setError(err instanceof ApiError ? err.message : 'Something went wrong')
    } finally {
      setIsLoading(false)
    }
  }, [])

  return { orders, isLoading, error, fetchOrders }
}
```

---

## Anti-Patterns to Avoid

```typescript
// ❌ Fetching in useEffect when a Server Component would do
useEffect(() => { fetch('/api/orders').then(...) }, [])

// ❌ Prop drilling more than 2 levels deep
<Page>
  <Layout>
    <Section user={user}>
      <Card user={user}>
        <Avatar user={user} />  // Just use context or pass id only

// ❌ Giant components — split after 100 lines
// ❌ Hardcoded API URLs — always use process.env.API_URL
// ❌ Boolean prop soup — use variants
<Button primary large rounded outlined />  // ❌
<Button variant="outline" size="lg" />    // ✅

// ❌ Unhandled promise rejections
fetch('/api/orders') // no await, no catch

// ❌ useEffect for derived state
const [fullName, setFullName] = useState('') // ❌ unnecessary state
useEffect(() => setFullName(`${first} ${last}`), [first, last])
const fullName = `${first} ${last}` // ✅ just compute it
```

---

## Import Order Convention

```typescript
// 1. React / Next.js core
import { useState, useEffect } from 'react'
import { redirect } from 'next/navigation'
import Image from 'next/image'

// 2. Third-party libraries
import { z } from 'zod'
import { useForm } from 'react-hook-form'

// 3. Internal — absolute paths via @/
import { apiFetch } from '@/lib/api'
import { Button } from '@/components/ui/Button'
import type { Order } from '@/types/order'

// 4. Relative (same feature only)
import { OrderRow } from './OrderRow'
```

---

## Notes

- **Co-locate tests** with the component they test
- **No magic numbers** — use named constants
- **Prefer composition over inheritance** in all component design
- **Keep pages thin** — pages should compose features, not contain logic
- **Use `satisfies` in TypeScript** for config objects to get inference + type safety
