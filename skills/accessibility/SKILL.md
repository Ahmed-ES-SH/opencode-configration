---
name: accessibility
description: Ensures web applications meet accessibility standards with semantic HTML, keyboard navigation, ARIA, and screen reader support. Use when the request involves a11y, keyboard navigation, screen readers, ARIA attributes, semantic HTML, focus management, or color contrast.
---

# Accessibility Skill

## Trigger

Use when the request mentions: **accessibility, a11y, keyboard navigation, screen readers, ARIA attributes, semantic HTML, focus management, color contrast, or accessible forms/navigation**.

---

## Scope

- Semantic HTML structure
- Keyboard navigation and focus management
- ARIA roles, labels, and live regions (only when HTML semantics fall short)
- Screen reader compatibility
- Color contrast and visual indicators
- Accessible forms with proper error association
- Skip links and landmark navigation
- Testing accessibility

---

## Core Rules

1. **Semantic HTML first** — Use the right element before reaching for ARIA
2. **ARIA is a last resort** — Don't add ARIA where native semantics work
3. **Every interactive element must be keyboard-reachable**
4. **Every image needs alt text** — empty `alt=""` for decorative images
5. **Every form input needs a visible, associated label**
6. **Color alone must not convey meaning**
7. **Focus must be visible at all times**

---

## Semantic Structure

```typescript
// ❌ Div soup
<div className="header">
  <div className="nav">
    <div onClick={() => navigate('/home')}>Home</div>
  </div>
</div>

// ✅ Semantic landmarks
<header>
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/home">Home</a></li>
      <li><a href="/orders">Orders</a></li>
    </ul>
  </nav>
</header>

<main id="main-content">
  <h1>Dashboard</h1>
  <section aria-labelledby="orders-heading">
    <h2 id="orders-heading">Recent Orders</h2>
    {/* content */}
  </section>
</main>

<footer>
  <p>© 2024 Company Name</p>
</footer>
```

---

## Skip Link (Required on Every App)

```typescript
// components/layout/SkipLink.tsx
export function SkipLink() {
  return (
    <a
      href="#main-content"
      className="sr-only focus:not-sr-only focus:fixed focus:top-4 focus:left-4 focus:z-50 focus:rounded focus:bg-primary focus:px-4 focus:py-2 focus:text-white focus:outline-none"
    >
      Skip to main content
    </a>
  )
}

// app/layout.tsx
<body>
  <SkipLink />
  <Header />
  <main id="main-content" tabIndex={-1}>
    {children}
  </main>
</body>
```

---

## Accessible Forms

```typescript
// ✅ Every input has an associated label
// ✅ Errors are associated with their input via aria-describedby
// ✅ Required fields are marked both visually and semantically

export function OrderForm() {
  const { register, formState: { errors } } = useForm<OrderFormData>()

  return (
    <form noValidate>
      <div>
        <label htmlFor="quantity" className="block text-sm font-medium">
          Quantity <span aria-hidden="true">*</span>
          <span className="sr-only">(required)</span>
        </label>
        <input
          id="quantity"
          type="number"
          {...register('quantity')}
          aria-required="true"
          aria-invalid={!!errors.quantity}
          aria-describedby={errors.quantity ? 'quantity-error' : undefined}
          className="mt-1 block w-full rounded border px-3 py-2
            focus:outline-none focus:ring-2 focus:ring-primary
            aria-[invalid=true]:border-destructive"
        />
        {errors.quantity && (
          <p id="quantity-error" role="alert" className="mt-1 text-xs text-destructive">
            {errors.quantity.message}
          </p>
        )}
      </div>
    </form>
  )
}
```

---

## Accessible Modal / Dialog

```typescript
// components/ui/Modal.tsx — Use native <dialog> or Radix UI Dialog
'use client'
import * as Dialog from '@radix-ui/react-dialog'

interface ModalProps {
  open: boolean
  onClose: () => void
  title: string
  children: React.ReactNode
}

export function Modal({ open, onClose, title, children }: ModalProps) {
  return (
    <Dialog.Root open={open} onOpenChange={onClose}>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 bg-black/50 data-[state=open]:animate-in" />
        <Dialog.Content
          className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 rounded-lg bg-background p-6 shadow-xl w-full max-w-md focus:outline-none"
          aria-describedby="modal-description"
        >
          <Dialog.Title className="text-lg font-semibold">{title}</Dialog.Title>
          <div id="modal-description">{children}</div>
          <Dialog.Close asChild>
            <button className="absolute top-4 right-4" aria-label="Close dialog">
              <XIcon aria-hidden="true" />
            </button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  )
}
// Radix handles: focus trap, Escape key, aria-modal, focus return on close
```

---

## Accessible Navigation (Active State)

```typescript
// components/layout/NavLink.tsx
'use client'
import Link from 'next/link'
import { usePathname } from 'next/navigation'
import { cn } from '@/lib/utils'

interface NavLinkProps {
  href: string
  children: React.ReactNode
}

export function NavLink({ href, children }: NavLinkProps) {
  const pathname = usePathname()
  const isActive = pathname === href || pathname.startsWith(`${href}/`)

  return (
    <Link
      href={href}
      aria-current={isActive ? 'page' : undefined}
      className={cn(
        'flex items-center gap-2 rounded px-3 py-2 text-sm transition-colors',
        isActive
          ? 'bg-primary text-primary-foreground font-medium'
          : 'hover:bg-muted'
      )}
    >
      {children}
    </Link>
  )
}
```

---

## Live Regions for Dynamic Content

```typescript
// Announce status changes to screen readers
export function StatusAnnouncer({ message }: { message: string }) {
  return (
    <div
      role="status"           // polite: waits for user to finish
      aria-live="polite"
      aria-atomic="true"
      className="sr-only"     // visually hidden, but read by screen readers
    >
      {message}
    </div>
  )
}

// Use aria-live="assertive" only for errors / critical alerts
<div role="alert" aria-live="assertive">
  {errorMessage}
</div>
```

---

## Focus Management

```typescript
// Manage focus when content changes dynamically
'use client'
import { useEffect, useRef } from 'react'

export function DynamicContent({ isLoaded }: { isLoaded: boolean }) {
  const headingRef = useRef<HTMLHeadingElement>(null)

  useEffect(() => {
    if (isLoaded) {
      headingRef.current?.focus()
    }
  }, [isLoaded])

  return isLoaded ? (
    <h2 ref={headingRef} tabIndex={-1} className="focus:outline-none">
      Results Loaded
    </h2>
  ) : null
}
```

---

## Accessibility Testing

```typescript
// Install axe-core for automated a11y checks
// npm install -D @axe-core/react jest-axe

import { axe, toHaveNoViolations } from 'jest-axe'
expect.extend(toHaveNoViolations)

it('has no accessibility violations', async () => {
  const { container } = render(<OrdersTable orders={mockOrders} />)
  const results = await axe(container)
  expect(results).toHaveNoViolations()
})
```

---

## Tailwind Focus Utilities

```css
/* Always use visible focus rings — never just remove them */
/* tailwind.config.ts */
ring-2 ring-primary ring-offset-2  /* ✅ Use this */
outline-none                        /* ❌ Never alone */
focus-visible:ring-2               /* ✅ Shows only on keyboard focus, not mouse */
```

---

## Notes

- **Use Radix UI or Headless UI** for complex interactive components (menus, dialogs, tabs) — they handle ARIA and keyboard correctly
- **Test with real assistive tech** — NVDA (Windows), VoiceOver (Mac/iOS)
- **Color contrast** — 4.5:1 for normal text, 3:1 for large text (WCAG AA)
- **Never rely on placeholder as a label** — it disappears on focus
- **`tabIndex={0}`** only to add focus to non-interactive elements when needed; **`tabIndex={-1}`** to programmatically focus without adding to tab order
