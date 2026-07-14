# Decision Guide: Which Tool for Which Scroll Animation

This walks through the reasoning in more depth than the SKILL.md summary table. Read this when the right tier isn't obvious from the quick table, or when a request mixes several effects and you need to decide whether to split them across tools or consolidate on one.

## The core question: does this need JavaScript at all?

Ask: **is the only thing happening a visual property change (opacity, transform, color, filter) driven purely by scroll position or viewport entry?**

If yes — no React state read, no side effect like firing an API call or playing a video, no complex multi-element choreography — native CSS can do it, full stop. This covers a surprising majority of real requests: hero fade-ins, card reveals, progress bars, simple parallax, sticky-then-fade headers.

If the animation needs to also trigger a React re-render, call a function, or coordinate timing across components in ways CSS keyframes can't express, you need JavaScript, and then the question becomes *how much* JavaScript.

## When native CSS is enough (Tier 1)

- Fade/slide/scale reveal when an element scrolls into view, playing once
- A single parallax layer moving at a different rate than scroll
- A scroll progress bar or reading-progress indicator
- Staggered entrance of a list/grid of siblings (via per-child `animation-delay`)
- A sticky header that shrinks or changes opacity as you scroll past a threshold

The only real downside is Firefox's rollout status for `animation-timeline` — it's implemented but may sit behind a flag for some users depending on version. Because the fallback is simply "no animation, element renders in its final state," this is safe to ship as progressive enhancement almost always. The one exception: if the client explicitly requires the animation to be pixel-identical across all browsers today (common in some agency contracts), lean Tier 2/3 instead.

## When you need Intersection Observer instead of / alongside CSS (Tier 2)

Reach for this when the "animation" is really a **state change with side effects**, not just a visual transition:

- Lazy-loading images/video only once they're near the viewport
- Firing an analytics event when a section becomes visible
- Playing/pausing a `<video>` element based on visibility
- Triggering a numeric counter animation that needs to run exactly once and hold its end value in React state
- Conditionally rendering different content based on scroll position (not just styling it differently)

Intersection Observer is cheap (main thread, but it doesn't run per-scroll-frame — it only fires on threshold crossings) and gives you a boolean or ratio you can put in state. Don't use it purely to drive a CSS transform when native CSS could do the same thing with zero JS — that's paying a JS cost for something the compositor could've handled for free.

## When Motion (Framer Motion) earns its bundle cost (Tier 3)

Motion is the right call when:

- The codebase is already React-heavy and the team wants animations expressed declaratively as component props rather than imperative timeline code
- You need `whileInView` (scroll-triggered) mixed with gesture animations (`whileHover`, `whileTap`) and exit animations (`AnimatePresence`) in the same component tree — Motion unifies all of these under one mental model
- You need `useScroll` + `useTransform` for scroll-linked values that also need to interact with React state or other component logic
- Layout animations (`layout` prop) need to coexist with scroll effects — this is a Motion-specific capability CSS/GSAP don't replicate as cleanly

Don't reach for Motion just to do a single fade-in-on-scroll — that's tier-1 territory and Motion's ergonomics don't outweigh the ~30KB cost for something CSS does for free. Motion earns its keep when several *different kinds* of animation (scroll + gesture + layout + exit) need to compose together.

## When GSAP + ScrollTrigger is worth it (Tier 4)

- **Pinning**: keeping a section fixed in the viewport while content scrolls/animates past it (scrollytelling, step-through product features). This is GSAP's strongest, most reliable use case — `pin: true` handles cross-browser quirks that are genuinely hard to replicate by hand.
- **Scrubbed timelines with many coordinated elements**: e.g. a hero where 8 different elements need to move, scale, and fade at different rates all tied to the same scroll range, with precise relative timing.
- **SVG path morphing or motion paths tied to scroll.**
- Sites where animation quality is the literal product (agency portfolios, awarded sites, product launch pages) and the team wants an imperative, timeline/DAW-style mental model.

The tradeoffs: heavier bundle than native CSS, imperative API (more manual cleanup — always `.kill()` timelines/ScrollTriggers on unmount), and some advanced plugins (SplitText, MorphSVG, ScrollSmoother) require a paid Club GSAP license for commercial use, though the core plus ScrollTrigger is free. Mention this licensing nuance to the user if their request would need a paid plugin.

## Mixing tools

It's completely normal for one page to use native CSS for simple reveals and GSAP for one hero scrollytelling sequence, or Motion for UI micro-interactions alongside a native-CSS progress bar. Don't force a single tool across an entire codebase if the requests genuinely differ in complexity — that's over-engineering the simple parts and under-powering the complex part. The one thing to avoid is using *two heavy JS libraries* for overlapping jobs (e.g. both Motion and GSAP driving scroll-linked values on the same page) — that's pure bundle waste and a recipe for animation conflicts.
