---
name: debugging
description: Diagnoses and resolves errors, bugs, crashes, and runtime issues in Next.js applications. Use when debugging hydration mismatches, Server/Client Component errors, build failures, auth problems, or API integration errors.
---

# Debugging Skill

## Trigger

Use when the request includes: **an error message, bug report, crash, stack trace, hydration mismatch, broken UI behavior, failing build, runtime error, or any "why is X not working" question**.

---

## Scope

- Hydration errors and mismatches
- Server Component vs Client Component errors
- Build failures and TypeScript errors
- Route and navigation issues
- Auth and session problems
- Fetch and API integration errors
- Infinite re-renders and stale closures
- Next.js-specific gotchas

---

## Hydration Errors

**Symptom:** `Error: Hydration failed because the initial UI does not match what was rendered on the server.`

**Cause:** Server-rendered HTML doesn't match what React tries to render on the client.

```typescript
// ❌ Common causes:

// 1. Browser-only APIs in Server Component
const theme = localStorage.getItem('theme') // fails on server

// 2. Date formatting that differs server vs client
<p>{new Date().toLocaleTimeString()}</p> // timezone mismatch

// 3. Math.random() or Date.now() in render
<div id={`item-${Math.random()}`} />

// ✅ Fix 1: Move to Client Component with useEffect
'use client'
import { useState, useEffect } from 'react'

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])
  if (!mounted) return null // or a skeleton
  return <>{children}</>
}

// ✅ Fix 2: suppressHydrationWarning for expected mismatches (time, date)
<time suppressHydrationWarning>{new Date().toLocaleTimeString()}</time>

// ✅ Fix 3: Use stable IDs (from data, not random)
<div id={`item-${item.id}`} />
```

---

## "use client" / "use server" Confusion

```typescript
// ❌ Importing a Server Component into a Client Component
// app/page.tsx (Server)
import ClientWrapper from './ClientWrapper' // has "use client"
// Inside ClientWrapper:
import ServerDataComponent from './ServerDataComponent' // ❌ breaks

// ✅ Pass Server Components as children/props
// ClientWrapper.tsx
'use client'
export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>
}

// app/page.tsx (Server)
<ClientWrapper>
  <ServerDataComponent /> {/* ✅ Server Component passed as prop */}
</ClientWrapper>

// ❌ Using hooks in Server Components
export default async function Page() {
  const [count, setCount] = useState(0) // Error: hooks not allowed in Server Components
}

// ✅ Extract the interactive part into a Client Component
'use client'
export function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

---

## Common Build Errors

```bash
# Error: Cannot find module '@/components/...'
# Fix: Check tsconfig.json paths and next.config.ts
{
  "compilerOptions": {
    "paths": { "@/*": ["./*"] }
  }
}

# Error: Type error: Property 'X' does not exist on type 'Session'
# Fix: Augment next-auth types
// types/next-auth.d.ts
import 'next-auth'
declare module 'next-auth' {
  interface Session {
    accessToken: string
    user: { id: string; role: string } & DefaultSession['user']
  }
  interface User {
    role: string
    accessToken: string
  }
}
declare module 'next-auth/jwt' {
  interface JWT {
    id: string
    role: string
    accessToken: string
  }
}

# Error: Server Actions must be async functions
export async function myAction(data: FormData) { ... } // ✅ always async
```

---

## Fetch / API Errors

```typescript
// Debugging fetch in Server Components
async function fetchData() {
  const res = await fetch(`${process.env.API_URL}/api/orders`, {
    cache: 'no-store',
  })

  // Always check res.ok before .json()
  if (!res.ok) {
    console.error('API Error:', res.status, res.statusText)
    const body = await res.text() // Sometimes JSON, sometimes HTML error page
    console.error('Body:', body)
    throw new Error(`API request failed: ${res.status}`)
  }

  return res.json()
}

// Verify env vars are available
console.log('API_URL:', process.env.API_URL) // undefined on client — expected
// Check in terminal output (server-side log), not browser console
```

---

## Infinite Re-renders

```typescript
// ❌ Object/array created in render as dependency
useEffect(() => {
  fetchData(filters) // filters is { status: 'active' } created inline
}, [filters]) // new object reference each render → infinite loop

// ✅ Fix: Memoize or use primitives as deps
const [status, setStatus] = useState('active')
useEffect(() => {
  fetchData({ status })
}, [status]) // primitive — stable reference

// ❌ Updating state unconditionally in useEffect
useEffect(() => {
  setData(processData(rawData)) // triggers re-render → loop
})

// ✅ Always add dependency array
useEffect(() => {
  setData(processData(rawData))
}, [rawData]) // only runs when rawData changes
```

---

## Authentication Debugging

```typescript
// Check session in Server Component
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'

export default async function ProtectedPage() {
  const session = await getServerSession(authOptions)
  console.log('Session:', JSON.stringify(session, null, 2)) // Check server logs

  if (!session) {
    redirect('/login') // No session → redirect
  }
}

// Check session in Client Component
'use client'
import { useSession } from 'next-auth/react'

export function DebugSession() {
  const { data: session, status } = useSession()
  console.log('Status:', status, 'Session:', session)
  // status: 'loading' | 'authenticated' | 'unauthenticated'
}

// Common auth issues:
// 1. NEXTAUTH_SECRET not set → sessions don't persist
// 2. NEXTAUTH_URL wrong → redirect loops
// 3. Callback URL mismatch → OAuth fails
```

---

## Next.js Specific Gotchas

```typescript
// 1. params in dynamic routes are now async in Next.js 15
// ❌ Old
export default function Page({ params }: { params: { id: string } }) { ... }

// ✅ New
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
}

// 2. cookies() and headers() are async in Next.js 15
import { cookies } from 'next/headers'
const cookieStore = await cookies()
const token = cookieStore.get('token')

// 3. useRouter().push() doesn't trigger loading.tsx
// Use router.refresh() after mutations to re-render Server Components

// 4. layout.tsx doesn't re-render on navigation between child routes
// Move dynamic content to page.tsx or use usePathname() in client layouts
```

---

## Debugging Checklist

```
□ Check browser console for client errors
□ Check terminal (server) for server-side errors
□ Verify .env.local has all required variables
□ Confirm API_URL is correct and accessible from server
□ Add console.log in Server Component — check terminal, not browser
□ Test API endpoint directly with curl or Postman
□ Check Network tab for failed requests
□ Verify session exists with getServerSession() log
□ Check if component should be "use client" or server
□ Look for async/await missing on server actions or route handlers
□ Verify TypeScript types match actual API response shape
```

---

## Notes

- **Server logs are in your terminal**, not the browser console
- **React DevTools** for component tree inspection and profiling
- **Next.js DevTools** (browser extension) for cache and route inspection
- When an error occurs in a Server Component and you have an `error.tsx`, it will catch it — but `error.tsx` must be a Client Component with `'use client'`
