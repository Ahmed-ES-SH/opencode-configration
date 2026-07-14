# GSAP + ScrollTrigger for React/Next.js

Reach for this tier specifically for pinned/scrollytelling sections, scrubbed timelines coordinating many elements, or SVG motion-path/morph effects tied to scroll — the cases where native CSS and Motion both fall short on cross-browser reliability or orchestration complexity. Requires a client component (`"use client"`).

## Setup: the `useGSAP` hook (do this, not raw `useEffect`)

`@gsap/react`'s `useGSAP` hook handles cleanup automatically and scopes selectors to a container, avoiding the classic GSAP-in-React memory leak (animations/ScrollTriggers surviving past unmount and stacking up on re-renders/navigation).

```bash
npm install gsap @gsap/react
```

```jsx
"use client";
import { useRef } from "react";
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger, useGSAP);

export function FeatureSection() {
  const containerRef = useRef(null);

  useGSAP(
    () => {
      const cards = gsap.utils.toArray(".feature-card");
      cards.forEach((card, i) => {
        gsap.from(card, {
          scrollTrigger: {
            trigger: card,
            start: "top 85%",
            toggleActions: "play none none reverse",
          },
          y: 60,
          opacity: 0,
          duration: 0.8,
          delay: i * 0.08,
          ease: "power3.out",
        });
      });
    },
    { scope: containerRef }
  );

  return (
    <div ref={containerRef} className="grid grid-cols-3 gap-8">
      {features.map((f) => (
        <div key={f.id} className="feature-card">
          {f.title}
        </div>
      ))}
    </div>
  );
}
```

`{ scope: containerRef }` means `.feature-card` selectors only match within this container — this is what lets you reuse the same component multiple times on a page without ScrollTrigger instances colliding.

## Pinning a section (scrollytelling)

```jsx
useGSAP(() => {
  const tl = gsap.timeline({
    scrollTrigger: {
      trigger: ".pin-wrapper",
      start: "top top",
      end: "+=2000", // pin lasts for 2000px of scroll
      pin: true,
      scrub: 1, // ties timeline progress to scroll, with 1s smoothing
    },
  });

  tl.to(".step-1", { opacity: 0, x: -100 })
    .to(".step-2-visual", { scale: 1.2 }, "<") // "<" = start at same time as previous
    .to(".step-2", { opacity: 1, x: 0 }, "<");
}, { scope: containerRef });
```

`scrub: true` (or a number for smoothing delay) is what makes this scroll-linked rather than scroll-triggered — the timeline's progress becomes a direct function of scroll position within the pinned range, rather than playing at its own pace once triggered.

## Cleanup — do not skip this

If you're not using `useGSAP` for some reason (you should be), you must manually kill both the timeline and any ScrollTrigger instances on unmount, or they'll keep listening to scroll on a detached DOM node:

```jsx
useEffect(() => {
  const tl = gsap.timeline({ scrollTrigger: { trigger: ref.current, /* ... */ } });
  return () => {
    tl.scrollTrigger?.kill();
    tl.kill();
  };
}, []);
```

`useGSAP` does this automatically via its own cleanup, which is the main reason to prefer it over hand-rolled `useEffect` + manual GSAP calls.

## Refreshing ScrollTrigger after layout changes

If content loads asynchronously (images, fetched data) and shifts layout after ScrollTrigger has already calculated its trigger positions, call `ScrollTrigger.refresh()` once the layout has settled (e.g. after images load, or in a `useEffect` that runs after data arrives). Stale trigger positions are one of the most common "this worked in dev but broke in prod" GSAP bugs, usually caused by content loading after ScrollTrigger's initial measurement.

## Performance notes

- **`scrub` with a numeric smoothing value** (e.g. `scrub: 0.5`) feels better than `scrub: true` (instant/1:1) for most scrubbed effects — it adds a small lag that reads as smoother rather than mechanically snapping to scroll position every frame.
- **Batch ScrollTrigger creation** with `ScrollTrigger.batch()` when animating many similar elements (e.g. a long list of cards all doing the same fade-in) rather than creating one ScrollTrigger per element in a loop — batching reduces overhead and lets you configure a single `onEnter` callback for the whole group.
- GSAP still benefits from the transform/opacity rule — animating `x`/`y`/`scale`/`opacity`/`rotation` keeps things on the fast path; avoid animating `width`/`height`/`top`/`left` with GSAP for the same reasons as everywhere else.
- **SSR note**: GSAP requires the DOM, so it only runs client-side. Server Components can render the static markup underneath; the animation attaches after hydration via the client component boundary. Don't try to import/register GSAP plugins at module scope in a file that might be imported by a Server Component — keep GSAP usage inside `"use client"` files only.

## Licensing note

GSAP core plus ScrollTrigger are free for commercial use. Some advanced plugins — SplitText, MorphSVGPlugin, ScrollSmoother, DrawSVGPlugin — require a paid "Club GSAP" license for commercial projects (they became bundled free for a period; check current licensing before assuming). If a request specifically needs text-splitting animations or SVG morphing, mention this to the user rather than assuming it's free, since licensing terms have shifted over time.
