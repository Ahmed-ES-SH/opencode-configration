---
name: documentation
description: Generates and maintains project documentation including READMEs, ADRs, component docs, and developer guides. Use when creating README content, technical documentation, handoff notes, component documentation, or implementation guides.
---

# Documentation Skill

## Trigger

Use when the request asks for: **README content, technical documentation, handoff notes, ADRs (Architecture Decision Records), component documentation, implementation explanations, or developer guides**.

---

## Scope

- Project README
- Component API documentation
- Architecture Decision Records (ADRs)
- Feature implementation guides
- Onboarding documentation
- API integration docs
- Environment setup guides
- Changelog entries

---

## Project README Template

```markdown
# Project Name

Brief description of what the app does and who it's for.

## Stack

| Layer      | Technology                     |
|------------|-------------------------------|
| Frontend   | Next.js 15 (App Router), TypeScript |
| Styling    | Tailwind CSS                  |
| Auth       | NextAuth.js v4                |
| Backend    | Laravel 11 REST API           |
| Deployment | Vercel                        |

## Prerequisites

- Node.js 20+
- npm 10+
- Access to the backend API (see [API Setup](#api-setup))

## Getting Started

```bash
# Clone
git clone https://github.com/yourorg/yourapp.git
cd yourapp

# Install dependencies
npm install

# Set up environment
cp .env.example .env.local
# Fill in required values (see Environment Variables section)

# Run development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## Environment Variables

| Variable            | Required | Description                          |
|---------------------|----------|--------------------------------------|
| `NEXTAUTH_SECRET`   | ✅       | JWT signing secret (min 32 chars)    |
| `NEXTAUTH_URL`      | ✅       | App URL (e.g. http://localhost:3000) |
| `API_URL`           | ✅       | Backend API base URL                 |
| `NEXT_PUBLIC_APP_URL` | ✅     | Public app URL                       |

Generate `NEXTAUTH_SECRET`: `openssl rand -base64 32`

## Project Structure

```
app/              # Next.js App Router
components/
  ui/             # Reusable primitives
  features/       # Feature-specific components
lib/              # Utilities, API client, auth config
hooks/            # Shared React hooks
types/            # TypeScript type definitions
validations/      # Zod schemas
```

## Available Scripts

| Command          | Description                        |
|------------------|------------------------------------|
| `npm run dev`    | Start development server           |
| `npm run build`  | Create production build            |
| `npm run start`  | Start production server            |
| `npm run lint`   | Run ESLint                         |
| `npm test`       | Run unit/component tests           |
| `npm run e2e`    | Run Playwright E2E tests           |

## Deployment

See [Deployment Guide](./docs/deployment.md) for Vercel and Docker instructions.
```

---

## Component Documentation

```typescript
/**
 * OrdersTable — displays a paginated list of orders with optional delete action.
 *
 * @example
 * // Server Component usage (preferred)
 * const orders = await apiFetch<Order[]>('/api/orders')
 * <OrdersTable orders={orders} />
 *
 * @example
 * // With delete capability (admin only)
 * <OrdersTable orders={orders} onDelete={handleDelete} />
 */
interface OrdersTableProps {
  /** Array of orders to display */
  orders: Order[]
  /** Called with order ID when delete button is clicked. Omit to hide delete buttons. */
  onDelete?: (id: string) => void
  /** Shows skeleton UI while data is loading */
  isLoading?: boolean
  /** Override default empty state message */
  emptyMessage?: string
}

export function OrdersTable({
  orders,
  onDelete,
  isLoading = false,
  emptyMessage = 'No orders found',
}: OrdersTableProps) {
  // ...
}
```

---

## Architecture Decision Record (ADR)

```markdown
# ADR-001: Use App Router over Pages Router

**Date:** 2024-01-15
**Status:** Accepted
**Deciders:** [Team leads]

## Context

Starting a new Next.js project. Need to choose between Pages Router (stable, widely documented)
and App Router (newer, Vercel-recommended, React Server Components).

## Decision

Use App Router as the default routing system.

## Rationale

- Server Components eliminate most client-side data fetching boilerplate
- Better performance by default (less JavaScript sent to browser)
- Native streaming with Suspense
- Recommended by Vercel and the Next.js team for new projects
- Better layout nesting without prop drilling

## Consequences

**Positive:**
- Better performance characteristics out of the box
- Cleaner data fetching patterns
- Co-located layouts and loading states

**Negative:**
- Smaller community knowledge base (fewer Stack Overflow answers)
- Some third-party libraries not yet fully compatible
- Team needs to learn new mental model (Server vs Client Components)

## Alternatives Considered

- **Pages Router:** More familiar, more resources, but missing Server Components
- **Remix:** Similar patterns but switching frameworks adds more risk
```

---

## Feature Implementation Guide

```markdown
# Feature: Order Management

## Overview

Allows authenticated users to view, create, and manage their orders.
Admin users can view and delete all orders.

## Routes

| Route             | Component          | Access      |
|-------------------|--------------------|-------------|
| `/orders`         | OrdersPage         | Authenticated |
| `/orders/new`     | NewOrderPage       | Authenticated |
| `/orders/[id]`    | OrderDetailPage    | Owner or Admin |

## Data Flow

```
User visits /orders
  → middleware.ts checks session (redirects to /login if missing)
  → app/(dashboard)/orders/page.tsx (Server Component)
  → apiFetch<Order[]>('/api/orders') with Bearer token
  → Laravel API: GET /api/orders (returns paginated orders)
  → OrdersTable rendered on server, HTML sent to browser
```

## API Endpoints (Laravel)

| Method | Endpoint          | Description              |
|--------|-------------------|--------------------------|
| GET    | /api/orders       | List orders (paginated)  |
| POST   | /api/orders       | Create order             |
| GET    | /api/orders/{id}  | Get single order         |
| DELETE | /api/orders/{id}  | Delete order (admin)     |

## Key Files

- `app/(dashboard)/orders/page.tsx` — Orders list page
- `app/(dashboard)/orders/[id]/page.tsx` — Order detail
- `components/features/orders/OrdersTable.tsx` — Main table component
- `app/orders/actions.ts` — Server Actions (create, delete)
- `validations/order.ts` — Zod schemas

## State Management

No client state needed. All data fetched server-side.
Mutations via Server Actions → revalidatePath('/orders') on success.
```

---

## Onboarding Guide

```markdown
# Developer Onboarding

## Day 1 Checklist

- [ ] Clone repo and run locally (`npm run dev`)
- [ ] Get API access (ask team lead for `.env.local` values)
- [ ] Read `AGENT.md` — understand the architecture
- [ ] Read `docs/architecture.md` — system overview
- [ ] Make a small change and open a PR

## Key Concepts to Understand

1. **App Router** — all routes live in `app/`. No `pages/` directory.
2. **Server vs Client Components** — default is Server. Add `'use client'` only when needed.
3. **Data fetching** — happens in Server Components using `apiFetch()` in `lib/api.ts`.
4. **Auth** — handled by NextAuth. Use `getServerSession()` on server, `useSession()` on client.
5. **Mutations** — done via Server Actions in `**/actions.ts` files.

## Code Review Process

1. Open PR against `main`
2. Ensure: `npm run build`, `npx tsc --noEmit`, `npm test` all pass
3. Request review from a senior dev
4. Squash merge after approval
```

---

## Notes

- **Write docs as you build** — update README when adding env vars or scripts
- **ADRs are permanent records** — never delete them, mark as Superseded instead
- **Component docs** — JSDoc on components used across multiple features; skip for one-off components
- **Keep docs close to code** — `docs/` folder for architecture, inline JSDoc for component APIs
