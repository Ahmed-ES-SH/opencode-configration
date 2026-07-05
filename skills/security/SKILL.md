---
name: security
description: Protects Next.js applications against common vulnerabilities and ensures secure data handling. Use when discussing authentication, sessions, cookies, JWT, secrets, environment variables, XSS, CSRF, input validation, or RBAC.
---

# Security Skill

## Trigger

Use when the request mentions: **authentication, sessions, cookies, JWT, secrets, environment variables, XSS, CSRF, input validation, data sanitization, authorization, or secure data flow**.

---

## Scope

- Environment variable handling and secret protection
- Auth session management (NextAuth, JWT)
- Input validation and sanitization
- XSS prevention
- CSRF protection
- HTTP security headers
- API route authorization
- Role-based access control (RBAC)
- Secure cookie configuration

---

## Environment Variables

```bash
# .env.local (never commit to git)
NEXTAUTH_SECRET=long-random-secret-32chars-min
NEXTAUTH_URL=http://localhost:3000
API_URL=https://api.yourapp.com       # Server-only (no NEXT_PUBLIC_)
DATABASE_URL=...                       # Server-only

# Only expose to client when absolutely necessary
NEXT_PUBLIC_APP_URL=https://yourapp.com
NEXT_PUBLIC_STRIPE_KEY=pk_live_...    # Only publishable keys
```

**Rule: If it doesn't need to run in the browser, never prefix with `NEXT_PUBLIC_`.**

```typescript
// ✅ Safe — server-side only
const token = process.env.API_SECRET_KEY // undefined in browser

// ❌ Dangerous — exposed in JS bundle
const token = process.env.NEXT_PUBLIC_API_SECRET_KEY
```

---

## NextAuth Secure Configuration

```typescript
// lib/auth.ts
import { NextAuthOptions } from 'next-auth'

export const authOptions: NextAuthOptions = {
  secret: process.env.NEXTAUTH_SECRET, // Required in production

  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },

  cookies: {
    sessionToken: {
      name: `__Secure-next-auth.session-token`,
      options: {
        httpOnly: true,    // Not accessible via JS
        sameSite: 'lax',   // CSRF protection
        path: '/',
        secure: process.env.NODE_ENV === 'production', // HTTPS only in prod
      },
    },
  },

  callbacks: {
    async jwt({ token, user, account }) {
      if (user) {
        token.id = user.id
        token.role = user.role
        token.accessToken = account?.access_token
      }
      return token
    },
    async session({ session, token }) {
      session.user.id = token.id as string
      session.user.role = token.role as string
      session.accessToken = token.accessToken as string
      return session
    },
  },

  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
}
```

---

## Middleware-Based Route Protection

```typescript
// middleware.ts
import { withAuth } from 'next-auth/middleware'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import type { JWT } from 'next-auth/jwt'

export default withAuth(
  function middleware(req: NextRequest & { nextauth: { token: JWT | null } }) {
    const { token } = req.nextauth
    const pathname = req.nextUrl.pathname

    // Role-based access
    if (pathname.startsWith('/admin') && token?.role !== 'admin') {
      return NextResponse.redirect(new URL('/dashboard?error=forbidden', req.url))
    }

    if (pathname.startsWith('/billing') && !['admin', 'billing'].includes(token?.role as string)) {
      return NextResponse.redirect(new URL('/dashboard', req.url))
    }

    return NextResponse.next()
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token,
    },
  }
)

export const config = {
  matcher: [
    '/(dashboard)/:path*',
    '/admin/:path*',
    '/billing/:path*',
  ],
}
```

---

## Input Validation (All Boundaries)

```typescript
// validations/order.ts
import { z } from 'zod'

export const CreateOrderSchema = z.object({
  productId: z.string().uuid('Invalid product ID'),
  quantity: z.number().int().positive().max(1000),
  notes: z.string().max(500).optional(),
  // Never trust client-provided prices — compute server-side
})

export type CreateOrderInput = z.infer<typeof CreateOrderSchema>

// Server Action — validate before any DB/API call
export async function createOrder(formData: FormData) {
  const session = await getServerSession(authOptions)
  if (!session) throw new Error('Unauthorized')

  const raw = {
    productId: formData.get('productId'),
    quantity: Number(formData.get('quantity')),
    notes: formData.get('notes') || undefined,
  }

  const parsed = CreateOrderSchema.safeParse(raw)
  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors }
  }

  // Safe to use parsed.data — fully validated and typed
  await apiFetch('/api/orders', {
    method: 'POST',
    body: JSON.stringify(parsed.data),
  })
}
```

---

## Route Handler Authorization

```typescript
// app/api/orders/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'

export async function DELETE(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)

  // 1. Authenticated?
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // 2. Authorized? (owns this resource or is admin)
  const order = await apiFetch<Order>(`/api/orders/${params.id}`)
  if (order.userId !== session.user.id && session.user.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  // 3. Validate param
  const uuidSchema = z.string().uuid()
  if (!uuidSchema.safeParse(params.id).success) {
    return NextResponse.json({ error: 'Invalid ID' }, { status: 400 })
  }

  await apiFetch(`/api/orders/${params.id}`, { method: 'DELETE' })
  return NextResponse.json({ success: true })
}
```

---

## Security Headers

```typescript
// next.config.ts
const securityHeaders = [
  { key: 'X-DNS-Prefetch-Control', value: 'on' },
  { key: 'X-Frame-Options', value: 'SAMEORIGIN' },        // Clickjacking
  { key: 'X-Content-Type-Options', value: 'nosniff' },    // MIME sniffing
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-eval' 'unsafe-inline'", // relax for Next.js
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https://cdn.yourapp.com",
      "connect-src 'self' https://api.yourapp.com",
    ].join('; '),
  },
]

export default {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

---

## CSRF Protection

```typescript
// Next.js Server Actions have built-in CSRF protection via Origin header validation
// Route Handlers: validate Origin for state-changing methods

export async function POST(req: NextRequest) {
  const origin = req.headers.get('origin')
  const host = req.headers.get('host')

  if (origin && !origin.includes(host ?? '')) {
    return NextResponse.json({ error: 'CSRF check failed' }, { status: 403 })
  }
  // ...
}
```

---

## XSS Prevention

```typescript
// Next.js JSX escapes by default — never use dangerouslySetInnerHTML with user content
// ❌ Dangerous
<div dangerouslySetInnerHTML={{ __html: userProvidedContent }} />

// ✅ If you must render HTML (e.g., rich text from CMS), sanitize first
import DOMPurify from 'isomorphic-dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
```

---

## Notes

- **Never log access tokens or passwords** — even in development
- **Rotate `NEXTAUTH_SECRET`** after any suspected compromise
- **Short-lived JWTs** — use refresh token rotation for long sessions
- **Rate limit API routes** — use `@upstash/ratelimit` or middleware-level limiting
- **Audit dependencies** — run `npm audit` in CI and fail on high severity
- **HTTPS always in production** — enforce via HSTS header
