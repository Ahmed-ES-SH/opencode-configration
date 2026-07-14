# Motion (Framer Motion) for Scroll Animations in React/Next.js

Package name on npm is still `framer-motion` in most existing codebases, though the library has rebranded to "Motion" (import from `motion/react` in newer setups, or `framer-motion` in older ones — check the project's `package.json` and match what's already installed rather than assuming). Everything below uses `motion/react`-style APIs; swap the import path if the project uses the older package name.

Requires a client component in Next.js (`"use client"` at the top of the file) since it needs refs, effects, and browser APIs.

## Scroll-triggered: `whileInView`

This is the tier-3 equivalent of the native CSS `view()` reveal — reach for it specifically when you also need variants, stagger orchestration across siblings, or want it unified with hover/tap/exit animations in the same component.

```jsx
"use client";
import { motion } from "motion/react";

function RevealCard({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 40 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, margin: "0px 0px -15% 0px", amount: 0.2 }}
      transition={{ duration: 0.6, ease: [0.16, 1, 0.3, 1] }}
    >
      {children}
    </motion.div>
  );
}
```

- `viewport.once: true` — fires once, doesn't replay on scroll-back (default recommendation per the taste guidance in the main skill).
- `viewport.margin` — same idea as Intersection Observer's `rootMargin`: negative bottom margin triggers the reveal a bit before the element is fully in frame.
- `viewport.amount` — fraction of the element that must be visible (`"some"`, `"all"`, or 0-1).

## Staggered children with variants

```jsx
"use client";
import { motion } from "motion/react";

const container = {
  hidden: {},
  visible: { transition: { staggerChildren: 0.08 } },
};

const item = {
  hidden: { opacity: 0, y: 30 },
  visible: { opacity: 1, y: 0, transition: { duration: 0.5, ease: [0.16, 1, 0.3, 1] } },
};

function StaggeredGrid({ items }) {
  return (
    <motion.div
      initial="hidden"
      whileInView="visible"
      viewport={{ once: true, amount: 0.2 }}
      variants={container}
      className="grid grid-cols-3 gap-6"
    >
      {items.map((it) => (
        <motion.div key={it.id} variants={item}>
          {it.content}
        </motion.div>
      ))}
    </motion.div>
  );
}
```

## Scroll-linked: `useScroll` + `useTransform`

For parallax and progress-bar style effects where the value must track scroll position directly (not just trigger once).

```jsx
"use client";
import { useRef } from "react";
import { motion, useScroll, useTransform } from "motion/react";

function ParallaxSection() {
  const ref = useRef(null);
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ["start end", "end start"], // track while section is anywhere in viewport
  });

  const y = useTransform(scrollYProgress, [0, 1], ["-15%", "15%"]);

  return (
    <div ref={ref} style={{ overflow: "hidden", position: "relative" }}>
      <motion.img src="/hero-bg.jpg" style={{ y }} alt="" />
    </div>
  );
}
```

`useScroll`/`useTransform` produce `MotionValue`s, which update outside React's render cycle — setting `style={{ y }}` does **not** cause a re-render on every scroll frame. This is the key performance property of Motion: don't undo it by also calling `useState` + a scroll listener to track the same thing. If you need the numeric value in actual React state for some other purpose, use `useMotionValueEvent` to subscribe, not a raw scroll listener.

## Page-wide scroll progress bar

```jsx
"use client";
import { motion, useScroll } from "motion/react";

function ScrollProgressBar() {
  const { scrollYProgress } = useScroll();
  return (
    <motion.div
      style={{
        scaleX: scrollYProgress,
        transformOrigin: "0%",
        position: "fixed",
        top: 0,
        left: 0,
        right: 0,
        height: 4,
        background: "linear-gradient(90deg, #6366f1, #a855f7)",
      }}
    />
  );
}
```

Note: if this is the *only* scroll effect on the page, native CSS (`references/native-css.md` Pattern 3) does this with zero JS and zero bundle cost — only reach for the Motion version if the page already needs Motion for other effects.

## Performance-specific gotchas in Motion

- **Don't wrap huge lists of items each in their own `useScroll`.** Each call sets up its own scroll listener infrastructure. For long lists, prefer one parent-level `useScroll` and derive per-item transforms from it, or fall back to CSS `animation-timeline: view()` per item instead.
- **`layoutScroll` prop**: Motion doesn't measure the scroll offset of every ancestor by default for performance reasons. If a `layout` animation lives inside a scrollable container and seems to miscalculate position, add `layoutScroll` to that scrollable ancestor.
- **`layoutRoot` prop**: similarly needed on `position: fixed` ancestors so layout animations account for page scroll correctly.
- **Prefer `transform`-backed values** (`x`, `y`, `scale`, `rotate`) over animating `width`/`height`/`top`/`left` directly, same rule as everywhere else in this skill. Motion will happily let you animate layout properties, but doing so opts out of the compositor-only fast path.
- **Memoize static children** of animated parents with `React.memo` if the parent re-renders frequently for unrelated reasons — Motion values updating don't cause re-renders themselves, but if something else in the tree does, unnecessary child re-renders can still add up.

## Bundle size reality check

Motion's core is roughly 30KB gzipped for full features (layout animations, gestures, `AnimatePresence`, scroll). That's comparable to or slightly larger than GSAP core + ScrollTrigger combined. It's worth it when you need the unified declarative API across scroll + gesture + layout + exit animations in one mental model — not simply because it's the most popular library. For a single isolated scroll effect, check whether native CSS (Tier 1) already covers it before importing Motion at all.
