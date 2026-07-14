# Intersection Observer for Scroll-Triggered React State

Use this tier when the effect needs to touch React state or trigger a side effect (not purely visual CSS), or when you need guaranteed cross-browser identical behavior today without pulling in Motion/GSAP. It's cheap: it doesn't run per-scroll-frame, it only fires callbacks on threshold crossings, so it's much lighter than a raw `scroll` event listener.

## Minimal custom hook (no dependency needed)

```jsx
"use client";
import { useEffect, useRef, useState } from "react";

function useInView({ threshold = 0.15, rootMargin = "0px 0px -10% 0px", once = true } = {}) {
  const ref = useRef(null);
  const [isInView, setIsInView] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true);
          if (once) observer.unobserve(el);
        } else if (!once) {
          setIsInView(false);
        }
      },
      { threshold, rootMargin }
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [threshold, rootMargin, once]);

  return { ref, isInView };
}
```

Usage:

```jsx
function RevealSection({ children }) {
  const { ref, isInView } = useInView();

  return (
    <div
      ref={ref}
      style={{
        opacity: isInView ? 1 : 0,
        transform: isInView ? "translateY(0)" : "translateY(40px)",
        transition: "opacity 0.6s cubic-bezier(0.16,1,0.3,1), transform 0.6s cubic-bezier(0.16,1,0.3,1)",
      }}
    >
      {children}
    </div>
  );
}
```

Notice this still only animates `opacity`/`transform` via CSS transition — Intersection Observer decides *when* to flip the state, but the actual animation stays compositor-friendly. Don't animate layout properties here just because you're now "in JS-land."

## When to trigger side effects instead of styling

```jsx
useEffect(() => {
  if (isInView) {
    videoRef.current?.play();
    analytics.track("section_viewed", { section: "pricing" });
  } else {
    videoRef.current?.pause();
  }
}, [isInView]);
```

This is the actual justification for reaching for this tier over native CSS — CSS can't call a function or update application state.

## `rootMargin` for "trigger a little early"

`rootMargin: "0px 0px -10% 0px"` shrinks the effective bottom edge of the viewport by 10%, so the callback fires when the element is 10% of the viewport height before it would otherwise be considered visible. This is how you get reveals to feel anticipatory rather than laggy (see the taste guidance in the main SKILL.md — trigger around 10-20% early).

## Performance notes specific to this tier

- **One observer per distinct threshold/rootMargin config, not one per element** if you're observing many similar elements — you can pass multiple elements to a single `IntersectionObserver` instance via multiple `.observe()` calls and branch on `entry.target` in the callback, rather than instantiating a new observer per list item. For a handful of elements this doesn't matter much, but for a long list (50+) it avoids unnecessary observer overhead.
- Always **unobserve or disconnect on unmount** (the cleanup function above does this) — leaked observers on unmounted components are a real memory leak source in long-lived SPA-style Next.js apps.
- If you're only using this to replicate a fade-in that native CSS `view()` could do without any JS, prefer native CSS instead (Tier 1) — reserve this tier for when you genuinely need the React-side callback.
