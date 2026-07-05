---
name: architecture
description: Designs and evaluates system architecture for Next.js App Router applications. Use when discussing system design, folder structure, feature boundaries, state ownership, component composition, or monorepo integration.
---

# Architecture Skill

## Trigger

Use when the request involves: **system design, folder structure, feature boundaries, state ownership, component composition strategy, monorepo integration, or app-wide scalability decisions**.

---

## Scope

This skill handles **structural and design decisions** for the Next.js application:

- App Router folder structure for large-scale apps
- Feature-based architecture and boundary design
- State management strategy (server state vs client state)
- Shared vs feature-local code ownership
- Monorepo layout (Turborepo)
- Module boundaries and dependency rules
- Route group strategy

---

## Recommended Folder Structure (Production Scale)

```
/
├── app/
│   ├── (auth)/                     # Unauthenticated routes
│   │   ├── login/
│   │   │   ├── page.tsx
│   │   │   └── loading.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (dashboard)/                # Protected routes
│   │   ├── layout.tsx              # Auth guard + dashboard shell
│   │   ├── page.tsx                # /dashboard
│   │   ├── orders/
│   │   │   ├── page.tsx
│   │   │   ├── loading.tsx
│   │   │   ├── error.tsx
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   │   ├── invoices/
│   │   └── settings/
│   ├── api/                        # Route Handlers (BFF layer)
│   │   ├── auth/[...nextauth]/route.ts
│   │   ├── orders/route.ts
│   │   └── webhooks/route.ts
│   ├── layout.tsx                  # Root layout
│   └── not-found.tsx
│
├── components/
│   ├── ui/                         # Dumb, reusable primitives
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Modal.tsx
│   │   └── index.ts                # Barrel export
│   ├── layout/                     # App shell components
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx
│   │   └── Footer.tsx
│   └── features/                   # Smart, feature-specific
│       ├── orders/
│       │   ├── OrdersTable.tsx
│       │   ├── OrderForm.tsx
│       │   ├── OrderCard.tsx
│       │   └── index.ts
│       └── invoices/
│
├── lib/
│   ├── auth.ts                     # NextAuth config
│   ├── api.ts                      # Central fetch wrapper
│   ├── utils.ts                    # Shared utilities (cn, formatters)
│   └── constants.ts                # App-wide constants
│
├── hooks/                          # Shared custom hooks
│   ├── useDebounce.ts
│   └── useLocalStorage.ts
│
├── store/                          # Client state (Zustand)
│   ├── useUIStore.ts               # Sidebar open/close, modals
│   └── useCartStore.ts
│
├── types/                          # Global TypeScript types
│   ├── api.ts                      # API response shapes
│   ├── auth.ts                     # Session augmentation
│   └── index.ts
│
├── validations/                    # Zod schemas (shared)
│   ├── order.ts
│   └── user.ts
│
└── middleware.ts                   # Auth + route protection
```

---

## State Ownership Strategy

```
                    ┌─────────────────────────────────────┐
                    │         State Decision Tree          │
                    └─────────────────────────────────────┘

Is the data fetched from the backend?
  └─ YES → Server Component (default) or React Query (if client-side refresh needed)

Is the data derived from props?
  └─ YES → Just compute it, no useState

Is the data shared across many unrelated components?
  └─ YES → Zustand (client global state)
  └─ NO  → useState / useReducer in the component

Is it form state?
  └─ YES → React Hook Form (local to form component)

Is it URL state (filters, pagination)?
  └─ YES → useSearchParams / nuqs
```

### Zustand for UI State Only

```typescript
// store/useUIStore.ts
import { create } from 'zustand'

interface UIState {
  sidebarOpen: boolean
  toggleSidebar: () => void
  activeModal: string | null
  openModal: (id: string) => void
  closeModal: () => void
}

export const useUIStore = create<UIState>()(set => ({
  sidebarOpen: true,
  toggleSidebar: () => set(s => ({ sidebarOpen: !s.sidebarOpen })),
  activeModal: null,
  openModal: (id) => set({ activeModal: id }),
  closeModal: () => set({ activeModal: null }),
}))
```

### React Query for Client-Side Server State

```typescript
// When you need client-side fetching (real-time, user-triggered refresh)
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

export function useOrders(page: number) {
  return useQuery({
    queryKey: ['orders', page],
    queryFn: () => apiFetch<PaginatedResponse<Order>>(`/api/orders?page=${page}`),
    staleTime: 60_000,
  })
}

export function useDeleteOrder() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (id: string) => apiFetch(`/api/orders/${id}`, { method: 'DELETE' }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['orders'] }),
  })
}
```

---

## Route Protection via Middleware

```typescript
// middleware.ts
import { withAuth } from 'next-auth/middleware'
import { NextResponse } from 'next/server'

export default withAuth(
  function middleware(req) {
    const { token } = req.nextauth
    const isAdminRoute = req.nextUrl.pathname.startsWith('/admin')

    if (isAdminRoute && token?.role !== 'admin') {
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
  matcher: ['/(dashboard)/:path*', '/admin/:path*'],
}
```

---

## Module Dependency Rules

```
app/          → can import from: components/, lib/, hooks/, store/, types/
components/   → can import from: lib/, hooks/, types/ — NEVER from app/
lib/          → can import from: types/ — NEVER from components/ or app/
hooks/        → can import from: lib/, store/, types/
store/        → can import from: types/, lib/ — NEVER from components/
```

**Violations to watch for:**
- `lib/` importing a component
- A UI primitive importing from `features/`
- `store/` subscribing to route events directly

---

## Monorepo with Turborepo (Optional)

```
/apps
  /web          ← Next.js app
  /admin        ← Separate Next.js admin panel

/packages
  /ui           ← Shared component library
  /types        ← Shared TypeScript types
  /config       ← Shared ESLint, Tailwind, TS configs
  /api-client   ← Shared API fetch wrapper + types
```

---

## Notes

- **Route groups** (`(folder)`) are the primary tool for layout isolation without affecting URLs
- **Parallel routes** (`@slot`) for dashboards with multiple independent panels
- **Intercepting routes** (`(.)path`) for modals that preserve background context
- **Never share mutable state between Server Components** — they run in isolation per request
- **Keep `app/` thin** — pages import features, features own their logic
