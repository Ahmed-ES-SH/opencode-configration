# Native CSS Scroll-Driven Animations

Zero JavaScript, runs on the compositor thread (not the main thread), so it stays smooth even while React is busy doing something else. This should be your default for the majority of scroll-triggered and simple scroll-linked effects. Works great in Next.js since it needs no client component boundary at all — it's just CSS.

## The two timeline types — don't confuse them

- **`scroll()`** — timeline progress tracks the scroll position of a scroller (the page or an overflow container) directly. Use for progress bars, parallax, anything that should track scroll 1:1.
- **`view()`** — timeline progress tracks how far an *element* has moved through its scrollport (0% = about to enter, 100% = fully exited). Use for "animate this element as it passes through view" — the vast majority of reveal/parallax-on-an-element effects.

## Gotcha: `animation-timeline` is not part of the `animation` shorthand

Declare the `animation` shorthand first, then `animation-timeline` on its own line — otherwise the shorthand resets it to `auto` and silently breaks the effect.

```css
.reveal {
  animation: fade-slide-up linear both;
  animation-timeline: view();
  animation-range: entry 0% cover 30%;
}
```

## Pattern 1: Fade + slide-up reveal (scroll-triggered feel via view())

```css
@keyframes fade-slide-up {
  from {
    opacity: 0;
    transform: translateY(40px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.reveal-on-scroll {
  animation: fade-slide-up ease-out both;
  animation-timeline: view();
  /* Starts animating once element is 0% into view, finishes by 30% covered */
  animation-range: entry 0% cover 30%;
}

@media (prefers-reduced-motion: reduce) {
  .reveal-on-scroll {
    animation: none;
    opacity: 1;
    transform: none;
  }
}
```

This plays once per scroll-through and will replay if scrolled back — that's inherent to scroll-driven animations (they're not "fire once" by default the way an Intersection Observer trigger is). If you need strictly-once behavior, use Tier 2 (Intersection Observer) or Tier 3/4 instead, since native scroll-driven animations are fundamentally scrubbed, not fire-once events. (True fire-once "scroll-triggered" CSS via `animation-trigger` exists in newer Chrome versions — see the note at the bottom of this file.)

## Pattern 2: Staggered grid/list reveal

Use `animation-delay` per child via `nth-child`, or better, an inline custom property set by whatever's rendering the list (works cleanly with React/Next.js since you can set `style={{ '--i': index }}` per item):

```css
.stagger-item {
  animation: fade-slide-up ease-out both;
  animation-timeline: view();
  animation-range: entry 0% cover 30%;
  animation-delay: calc(var(--i, 0) * 80ms);
}
```

```jsx
{items.map((item, i) => (
  <div key={item.id} className="stagger-item" style={{ '--i': i }}>
    {item.content}
  </div>
))}
```

## Pattern 3: Scroll progress bar

```css
@keyframes grow-progress {
  from { transform: scaleX(0); }
  to { transform: scaleX(1); }
}

.progress-bar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 4px;
  transform-origin: left center;
  background: linear-gradient(90deg, #6366f1, #a855f7);
  animation: grow-progress auto linear;
  animation-timeline: scroll(root block);
}
```

`scroll(root block)` ties this to the root scroller's vertical (block-axis) progress. No JS, no `scroll` event listener, no `requestAnimationFrame`.

## Pattern 4: Parallax layer

```css
.parallax-bg {
  animation: parallax-move linear both;
  animation-timeline: view();
  animation-range: cover 0% cover 100%;
}

@keyframes parallax-move {
  from { transform: translateY(-15%); }
  to   { transform: translateY(15%); }
}
```

Keep parallax subtle (10-20% translation range) — large parallax offsets are the most common source of visible layout popping/edge-clipping bugs, since the element needs extra height/overflow margin to cover the full translated range without showing gaps.

## Pattern 5: Sticky-then-fade header

Combine `position: sticky` (no timeline needed) with a `view()`-driven opacity/blur fade as the user scrolls further past it:

```css
.site-header {
  position: sticky;
  top: 0;
  animation: header-shrink linear both;
  animation-timeline: scroll(root block);
  animation-range: 0px 200px;
}

@keyframes header-shrink {
  from { padding-block: 24px; box-shadow: none; }
  to   { padding-block: 12px; box-shadow: 0 2px 12px rgb(0 0 0 / 0.08); }
}
```

Note: animating `padding` here isn't compositor-only (it affects layout), which is fine for a single header element firing on scroll, but avoid this pattern on more than one or two elements per page — prefer a `transform: scale()` based shrink if you want this to be fully compositor-friendly.

## Browser support note (mid-2026)

`animation-timeline`/`view()`/`scroll()` are well-supported in Chromium and Safari. Firefox has implemented the spec but it may still be behind a flag depending on the user's version — treat this as progressive enhancement, not a hard dependency. The degradation path is graceful: unsupported browsers simply render the element in its resting/final CSS state with no animation, so nothing breaks visually, it's just static. Confirm this is acceptable to the person for their audience; if pixel-perfect cross-browser parity is a hard requirement, use Tier 2/3 instead.

Newer **`animation-trigger`** (true fire-once-and-hold scroll-triggered animations, as opposed to scrubbed `view()`/`scroll()` timelines) is much newer and currently Chrome-only. Only use it behind a `@supports` check with an Intersection Observer fallback for other browsers:

```css
@supports (animation-trigger: --t play-forwards) {
  .reveal-once {
    animation-trigger: --t play-forwards;
  }
}
```

Don't build a whole feature's only implementation on `animation-trigger` yet — it's a nice-to-have progressive enhancement layer on top of a `view()`-based or Intersection-Observer-based baseline, not a replacement for one.
