---
name: scroll-animations
description: Build scroll-based animations (fade-ins, reveals, parallax, scroll-linked progress bars, pinned/sticky sections, staggered lists) in React and Next.js that stay smooth at 60fps and don't tank Core Web Vitals. Use this skill whenever the user asks for "scroll animation," "animate on scroll," "reveal on scroll," "fade in as you scroll," "parallax," "scroll-triggered," "sticky/pinned section," "scroll progress bar," or wants a landing page / portfolio / marketing site to "feel alive" or "premium" while scrolling. Also use it any time an existing scroll animation is described as janky, laggy, stuttering, dropping frames, or hurting Lighthouse/CLS/INP scores — this skill covers the performance-safe fix, not just the visual effect. Covers native CSS scroll-driven animations, Motion (Framer Motion), GSAP ScrollTrigger, and Intersection Observer, with clear guidance on which to reach for and which to avoid.
---

# Scroll Animations for React & Next.js

Scroll animation requests almost always have two competing goals in tension: the person wants it to look **impressive**, and they need it to stay **fast**. Most janky, battery-draining scroll effects on the web come from a small set of repeated mistakes, not from the ideas being unreasonable. This skill exists to help you get both at once, and it leans on a 2026 landscape where the fastest option is often *not* the biggest library, it's the browser itself.

## Step 1: Classify the request before writing any code

Every scroll animation falls into one of two categories, and confusing them is the single biggest source of wasted effort:

- **Scroll-triggered** — something plays once (or reverses) when it enters/leaves the viewport. Fade-ins, slide-up reveals, staggered card grids, counters that start counting. The animation has its own timing/easing; scroll position just flips a switch.
- **Scroll-linked (scrubbed)** — the animation's progress is *directly* mathematically tied to scroll position. Parallax layers, progress bars, pinned sections that scrub through steps, scale/rotate effects that track exactly how far you've scrolled. There's no independent duration; scroll IS the timeline.

Say out loud (in your plan, not necessarily to the user) which one each requested effect is. It changes everything downstream: the CSS/API you reach for, whether Intersection Observer is even relevant, and how you keep it performant.

## Step 2: Pick the right tool — smallest capable option wins

Reach for tools in this order. Each step up costs bundle size and complexity, so only move to the next tier when the current one genuinely can't do what's being asked.

| Tier | Tool | Reach for it when | Cost |
|---|---|---|---|
| 1 | **Native CSS** (`animation-timeline`, `@scroll-timeline`, `view()`/`scroll()`) | Fades, reveals, progress bars, simple parallax, staggered grids — the majority of requests | 0 KB, runs on the compositor thread, immune to main-thread jank |
| 2 | **Intersection Observer** (vanilla or a ~1KB hook) | Need to trigger React state/side-effects on visibility (lazy-load, fire analytics, play/pause video) rather than purely visual animation | ~0-1 KB, main thread but very cheap |
| 3 | **Motion (Framer Motion)** | Already React-heavy codebase, need `whileInView`/`useScroll`/`useTransform` ergonomics, gesture support, or exit animations alongside scroll effects | ~30 KB gzipped |
| 4 | **GSAP + ScrollTrigger** | Pinned multi-step sequences, scroll-scrubbed timelines with many coordinated elements, SVG morphing, or anything needing frame-perfect orchestration across dozens of elements | ~23 KB core + ~7 KB ScrollTrigger |

Don't default to GSAP or Framer Motion just because they're famous. A three-section fade-in-on-scroll landing page done in native CSS will outperform the same thing done in a JS library, with less code and zero bundle cost. Reserve tier 3/4 for when the person actually needs what only they provide: React-driven state changes, complex pinning, or multi-layer scrubbed timelines with many moving parts.

**Browser support caveat (as of mid-2026):** native `animation-timeline`/`view()` scroll-driven animations are well-supported in Chromium and Safari; Firefox has it implemented but it may still be gated behind a flag for some users. The graceful degradation is good — unsupported browsers just show the element in its resting/final state with no animation, so it's safe to ship as progressive enhancement. If the person needs guaranteed identical behavior on every browser today, prefer tier 2/3 instead, or pair native CSS with a JS fallback. Newer **scroll-triggered** (not scroll-linked) CSS via `animation-trigger` is Chrome-only and newer still — treat it as a progressive enhancement with an Intersection Observer fallback, never as the only implementation.

If you're unsure which tier fits, read `references/decision-guide.md` for a fuller walkthrough with concrete examples, and read the tier-specific reference file for implementation details before writing code:
- `references/native-css.md` — scroll-timeline, view(), animation-range, staggering, pinned/sticky sections in pure CSS
- `references/intersection-observer.md` — the lean vanilla/hook pattern for trigger-driven React state
- `references/framer-motion.md` — whileInView, useScroll, useTransform, layout animation caveats
- `references/gsap-scrolltrigger.md` — useGSAP setup, ScrollTrigger config, pinning, cleanup

## Step 3: The performance rules that actually matter

These apply no matter which tier you land on. Most "heavy" scroll animations are heavy because one of these was skipped, not because animation itself is expensive.

**Animate only `transform` and `opacity`.** These are the only two CSS property groups the browser can animate on the compositor thread without triggering layout (reflow) or paint. Animating `width`, `height`, `top`, `left`, `margin`, box-shadow spread, or filter blur forces the main thread to recalculate layout on every frame — that's where jank comes from. If you want something to "grow," scale it with `transform: scale()` instead of changing `width`/`height`. If you want it to "move down," use `transform: translateY()`, never `top` or `margin-top`.

**Never poll scroll position with a bare `scroll` event listener.** A naive `window.addEventListener('scroll', fn)` that reads layout properties (`getBoundingClientRect`, `offsetTop`, `scrollY`) and then writes styles in the same tick causes layout thrashing — forced synchronous reflow, repeated dozens of times per second. If you must hand-roll scroll tracking (rare — prefer tiers 1-4 above which handle this internally), always read all layout values first, then write inside a single `requestAnimationFrame`, and use `{ passive: true }` on the listener.

**Use `will-change` sparingly and remove it when idle.** `will-change: transform` tells the browser to promote an element to its own GPU layer ahead of time — useful for an element about to animate, harmful if left on dozens of elements permanently, since each promoted layer costs GPU memory. Apply it right before the animation starts (or via a class toggled on-scroll-into-view) and remove it once the animation finishes, rather than baking it into a global stylesheet rule that matches everything with `.animate-on-scroll`.

**Cap how many elements animate scroll-linked (scrubbed) effects simultaneously.** Scroll-triggered fade-ins that fire once are cheap even by the dozen. Scroll-linked/scrubbed effects (parallax, scrubbed timelines) that recompute every single scroll frame are the expensive category — if a design calls for 20+ simultaneously parallaxing elements, that's a strong signal to consolidate layers (e.g. a few parallax bands instead of per-element parallax) or move to GSAP/CSS native, which are built to handle this cheaply, rather than hand-rolled `useScroll`/`useTransform` chains on every item.

**Respect `prefers-reduced-motion`.** Always provide a reduced/no-motion path — wrap keyframes in `@media (prefers-reduced-motion: no-preference)`, or check `window.matchMedia('(prefers-reduced-motion: reduce)')` before starting JS-driven animations. This isn't just an accessibility nicety; large motion effects can trigger vestibular discomfort for some users, and it's a near-zero-cost thing to include from the start rather than retrofit.

**In Next.js: keep scroll-animated components client-only, and don't blow up the bundle.** Anything using GSAP, Framer Motion, or refs/`useEffect` needs `"use client"`. Server Components can render the static markup; the animation attaches after hydration. Don't import GSAP or Framer Motion into a component tree that's mostly server-rendered content just for one small effect — isolate the animated piece into its own client component so the rest of the page stays server-rendered and light. For native CSS scroll animations, no client boundary is even needed since it's pure CSS.

## Step 4: Make it actually look good, not just performant

Performance rules don't specify taste. A few defaults that consistently read as "premium" rather than "generic AOS-library fade":

- **Easing over linear.** Almost nothing in nature moves at a constant rate. Use `cubic-bezier` eases (CSS: `ease-out`, `cubic-bezier(0.16, 1, 0.3, 1)` for a nice "decelerate" feel; GSAP: `power3.out`, `power2.inOut`; Framer Motion: `ease: [0.16, 1, 0.3, 1]` or spring configs). Reserve `linear` for things that are literally scroll-scrubbed 1:1 (progress bars), where linear is correct because it must track scroll exactly.
- **Small distances, short durations.** Reveal animations that translate 60-100px over 0.5-0.8s read as elegant. Translating 300px+ or taking 1.5s+ reads as slow and try-hard. Same with fades: going from `opacity: 0` is fine, but combine it with a subtle transform rather than a pure fade for more visual interest.
- **Stagger, don't synchronize.** When multiple siblings (cards, list items, nav links) animate in together, offset each by roughly 60-120ms. Simultaneous identical animations on multiple elements read as static/robotic; staggered ones read as choreographed.
- **Trigger a little early.** Fire the reveal when the element is ~10-20% into the viewport (e.g. `start: "top 85%"` in ScrollTrigger, or `margin: "0px 0px -15% 0px"` in Framer Motion's viewport options) rather than waiting for it to be fully visible. Waiting for full visibility makes reveals feel like they're chasing the user's scroll instead of anticipating it.
- **Once vs. every time.** Default to triggering entrance reveals only once (`once: true` / `toggleActions: "play none none none"`). Re-triggering every time the user scrolls back up over a section usually feels chaotic rather than delightful — reserve replay-on-re-entry for cases where the person explicitly wants that (e.g. a playful micro-interaction), not for standard content reveals.

## Common request patterns → where to start

- "Fade in sections as I scroll down" → Tier 1 (native CSS) or Tier 2 if React state is also needed. See `references/native-css.md`.
- "Parallax hero image" → Tier 1 native CSS `animation-timeline: view()` for a single layer; Tier 3/4 if multiple coordinated layers.
- "Scroll progress bar at top of page" → Tier 1, trivial in pure CSS (`scroll-timeline: root`), see `references/native-css.md`.
- "Pin this section while content scrolls past it" / "scrollytelling" → Tier 4 (GSAP ScrollTrigger `pin: true`) is the most reliable cross-browser today. See `references/gsap-scrolltrigger.md`.
- "Stagger these cards in as they appear" → Tier 1 CSS with `animation-delay` per child, or Tier 3 Framer Motion `variants` + `staggerChildren`.
- "This scroll animation is laggy/janky" → Go straight to Step 3 above and audit for the transform/opacity rule, scroll-listener thrashing, and `will-change` misuse before touching the visual design.

Read the relevant reference file now before writing code — each contains full working examples, the exact hook/plugin setup, and gotchas specific to that tool.
