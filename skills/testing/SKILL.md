---
name: testing
description: Provides testing strategies, setup, and patterns for Next.js applications. Use when discussing unit tests, integration tests, component tests, E2E tests, mocking, fixtures, coverage, or regression validation.
---

# Testing Skill

## Trigger

Use when the request asks about: **unit tests, integration tests, component tests, E2E tests, mocking, fixtures, test setup, coverage strategy, or regression validation in Next.js**.

---

## Scope

- Unit testing with Jest + TypeScript
- Component testing with React Testing Library
- Server Action and Route Handler testing
- E2E testing with Playwright
- Mocking fetch, NextAuth sessions, and next/navigation
- Test data factories and fixtures
- CI test strategy

---

## Setup

```bash
# Jest + RTL
npm install -D jest jest-environment-jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event ts-jest

# Playwright
npm install -D @playwright/test
npx playwright install
```

```typescript
// jest.config.ts
import type { Config } from 'jest'
import nextJest from 'next/jest.js'

const createJestConfig = nextJest({ dir: './' })

const config: Config = {
  testEnvironment: 'jsdom',
  setupFilesAfterFramework: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/$1' },
  coverageThreshold: {
    global: { branches: 70, functions: 80, lines: 80 },
  },
}

export default createJestConfig(config)
```

```typescript
// jest.setup.ts
import '@testing-library/jest-dom'
```

---

## Component Tests (RTL)

```typescript
// components/features/orders/OrdersTable.test.tsx
import { render, screen, within } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { OrdersTable } from './OrdersTable'
import { mockOrders } from '@/tests/fixtures/orders'

describe('OrdersTable', () => {
  it('renders orders correctly', () => {
    render(<OrdersTable orders={mockOrders} />)
    expect(screen.getByText(mockOrders[0].id)).toBeInTheDocument()
    expect(screen.getAllByRole('row')).toHaveLength(mockOrders.length + 1) // +1 header
  })

  it('shows empty state when no orders', () => {
    render(<OrdersTable orders={[]} />)
    expect(screen.getByText(/no orders found/i)).toBeInTheDocument()
  })

  it('shows skeleton while loading', () => {
    render(<OrdersTable orders={[]} isLoading />)
    expect(screen.getByTestId('orders-skeleton')).toBeInTheDocument()
  })

  it('calls onDelete when delete button clicked', async () => {
    const onDelete = jest.fn()
    render(<OrdersTable orders={mockOrders} onDelete={onDelete} />)
    await userEvent.click(screen.getAllByRole('button', { name: /delete/i })[0])
    expect(onDelete).toHaveBeenCalledWith(mockOrders[0].id)
  })
})
```

---

## Mocking next/navigation and next-auth

```typescript
// Mock next/navigation
jest.mock('next/navigation', () => ({
  useRouter: () => ({ push: jest.fn(), replace: jest.fn(), refresh: jest.fn() }),
  usePathname: () => '/dashboard',
  useSearchParams: () => new URLSearchParams(),
  redirect: jest.fn(),
}))

// Mock next-auth session
jest.mock('next-auth/react', () => ({
  useSession: () => ({
    data: { user: { name: 'Test User', email: 'test@test.com' }, accessToken: 'mock-token' },
    status: 'authenticated',
  }),
  signIn: jest.fn(),
  signOut: jest.fn(),
}))

// Mock fetch globally
global.fetch = jest.fn()

beforeEach(() => {
  ;(global.fetch as jest.Mock).mockResolvedValue({
    ok: true,
    json: async () => ({ data: mockOrders, meta: { total: 10 } }),
  })
})

afterEach(() => jest.clearAllMocks())
```

---

## Server Action Tests

```typescript
// app/orders/actions.test.ts
import { createOrder } from './actions'

jest.mock('next-auth', () => ({
  getServerSession: jest.fn().mockResolvedValue({
    accessToken: 'mock-token',
  }),
}))

jest.mock('next/cache', () => ({ revalidatePath: jest.fn() }))

describe('createOrder', () => {
  it('returns error for invalid input', async () => {
    const formData = new FormData()
    formData.set('productId', 'not-a-uuid')
    formData.set('quantity', '-1')

    const result = await createOrder(formData)
    expect(result.error).toBeDefined()
  })

  it('creates order and revalidates path', async () => {
    ;(global.fetch as jest.Mock).mockResolvedValueOnce({ ok: true, json: async () => ({}) })

    const formData = new FormData()
    formData.set('productId', '550e8400-e29b-41d4-a716-446655440000')
    formData.set('quantity', '2')

    const result = await createOrder(formData)
    expect(result.success).toBe(true)
    expect(require('next/cache').revalidatePath).toHaveBeenCalledWith('/orders')
  })
})
```

---

## Test Fixtures

```typescript
// tests/fixtures/orders.ts
import type { Order } from '@/types'

export const mockOrder = (overrides?: Partial<Order>): Order => ({
  id: '550e8400-e29b-41d4-a716-446655440000',
  status: 'pending',
  total: 150.00,
  createdAt: '2024-01-15T10:00:00Z',
  customer: { id: '1', name: 'John Doe', email: 'john@example.com' },
  ...overrides,
})

export const mockOrders = Array.from({ length: 5 }, (_, i) =>
  mockOrder({ id: `order-${i + 1}`, total: (i + 1) * 50 })
)
```

---

## Playwright E2E Tests

```typescript
// e2e/orders.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Orders', () => {
  test.beforeEach(async ({ page }) => {
    // Set auth cookie / storage state
    await page.goto('/login')
    await page.fill('[name="email"]', 'admin@test.com')
    await page.fill('[name="password"]', 'password')
    await page.click('button[type="submit"]')
    await expect(page).toHaveURL('/dashboard')
  })

  test('displays orders list', async ({ page }) => {
    await page.goto('/orders')
    await expect(page.getByRole('table')).toBeVisible()
    await expect(page.getByRole('row')).toHaveCount.greaterThan(1)
  })

  test('creates a new order', async ({ page }) => {
    await page.goto('/orders/new')
    await page.selectOption('[name="productId"]', { label: 'Product A' })
    await page.fill('[name="quantity"]', '3')
    await page.click('button[type="submit"]')
    await expect(page).toHaveURL('/orders')
    await expect(page.getByRole('alert')).toContainText('Order created')
  })
})
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:3000',
    storageState: 'e2e/.auth/user.json', // reuse login state
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

---

## Notes

- **Test behavior, not implementation** — test what the user sees, not internal state
- **Never test implementation details** — no `wrapper.state()`, no direct hook calls outside components
- **Auth state in E2E** — use `storageState` to reuse login, don't re-login every test
- **MSW for API mocking** — use Mock Service Worker for integration tests involving real fetch logic
- **Coverage thresholds** — enforce in CI to prevent regressions; aim for 80%+ on critical paths
