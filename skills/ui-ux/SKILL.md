---
name: ui-ux
description: Handles visual design, layout, and interaction patterns for Next.js applications. Use when discussing layout, spacing, responsive design, typography, animations, component polish, Tailwind/CSS, or design system consistency.
---

# UI-UX Skill

## Trigger

Use when the request is about: **layout, spacing, responsive design, typography, interaction design, animations, component polish, visual hierarchy, design-system consistency, or Tailwind/CSS implementation**.

---

## Scope

This skill handles the **visual and interactive layer** of the Next.js application:

- Component design and composition
- Responsive layout with Tailwind CSS
- Typography systems and spacing scales
- Motion and animation (Framer Motion / CSS transitions)
- Design system enforcement (tokens, variants)
- Loading states, skeletons, empty states, error UI
- Form UX and validation feedback
- Dark mode implementation

---

## Layout System

### App Shell Pattern

```typescript
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className="bg-background text-foreground font-sans antialiased">
        <div className="min-h-screen flex flex-col">
          <Header />
          <main className="flex-1 container mx-auto px-4 py-8 max-w-7xl">
            {children}
          </main>
          <Footer />
        </div>
      </body>
    </html>
  )
}
```

### Dashboard Sidebar Layout

```typescript
// app/(dashboard)/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen overflow-hidden">
      <Sidebar className="w-64 shrink-0 hidden md:flex" />
      <div className="flex-1 flex flex-col overflow-auto">
        <TopNav />
        <main className="flex-1 p-6">{children}</main>
      </div>
    </div>
  )
}
```

---

## Component Patterns

### Variant-based Component (cva)

```typescript
// components/ui/Button.tsx
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4',
        lg: 'h-12 px-6 text-base',
      },
    },
    defaultVariants: { variant: 'default', size: 'md' },
  }
)

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  loading?: boolean
}

export function Button({ className, variant, size, loading, children, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      disabled={loading || props.disabled}
      {...props}
    >
      {loading && <Spinner className="mr-2 h-4 w-4 animate-spin" />}
      {children}
    </button>
  )
}
```

### Skeleton Loader

```typescript
// components/skeletons/CardSkeleton.tsx
export function CardSkeleton() {
  return (
    <div className="rounded-lg border bg-card p-6 animate-pulse space-y-3">
      <div className="h-4 bg-muted rounded w-1/3" />
      <div className="h-3 bg-muted rounded w-full" />
      <div className="h-3 bg-muted rounded w-2/3" />
    </div>
  )
}
```

---

## Responsive Design

```typescript
// Tailwind responsive breakpoints
// sm: 640px | md: 768px | lg: 1024px | xl: 1280px | 2xl: 1536px

// Grid that adapts from 1 → 2 → 3 columns
<div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
  {items.map(item => <Card key={item.id} {...item} />)}
</div>

// Hide on mobile, show on desktop
<Sidebar className="hidden lg:flex" />

// Stack on mobile, row on desktop
<div className="flex flex-col md:flex-row gap-4">
```

---

## Animation (Framer Motion)

```typescript
// components/ui/FadeIn.tsx
'use client'
import { motion } from 'framer-motion'

export function FadeIn({ children, delay = 0 }: { children: React.ReactNode; delay?: number }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 12 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3, delay, ease: 'easeOut' }}
    >
      {children}
    </motion.div>
  )
}

// Staggered list
export function StaggeredList({ items }: { items: React.ReactNode[] }) {
  return (
    <motion.ul
      initial="hidden"
      animate="visible"
      variants={{ visible: { transition: { staggerChildren: 0.07 } } }}
    >
      {items.map((item, i) => (
        <motion.li
          key={i}
          variants={{ hidden: { opacity: 0, x: -10 }, visible: { opacity: 1, x: 0 } }}
        >
          {item}
        </motion.li>
      ))}
    </motion.ul>
  )
}
```

---

## Dark Mode

```typescript
// tailwind.config.ts
export default {
  darkMode: 'class', // toggle via <html class="dark">
  // ...
}

// components/ThemeToggle.tsx
'use client'
import { useTheme } from 'next-themes'

export function ThemeToggle() {
  const { theme, setTheme } = useTheme()
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      {theme === 'dark' ? <SunIcon /> : <MoonIcon />}
    </button>
  )
}
```

---

## Form UX with React Hook Form + Zod

```typescript
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Minimum 8 characters'),
})

type FormData = z.infer<typeof schema>

export function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
  })

  const onSubmit = async (data: FormData) => {
    // call server action or API
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <input {...register('email')} placeholder="Email"
          className="w-full rounded border px-3 py-2 text-sm focus:ring-2 focus:ring-primary" />
        {errors.email && <p className="mt-1 text-xs text-destructive">{errors.email.message}</p>}
      </div>
      <Button type="submit" loading={isSubmitting}>Sign In</Button>
    </form>
  )
}
```

---

## Notes

- **Use `cn()` utility** for conditional class merging (clsx + tailwind-merge)
- **Avoid inline styles** — always use Tailwind or CSS variables
- **Design tokens** — define colors/spacing in `tailwind.config.ts`, not hardcoded hex values
- **Loading UI** — every async page needs a `loading.tsx` or `<Suspense>` fallback
- **Empty states** — always design what happens when a list is empty
- **Error UI** — every data boundary needs an `error.tsx` with a retry action
