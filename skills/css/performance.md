# Performance Optimization

Render pipeline, layout shift, fonts, loading. The frame-budget rule (16.7ms, animate transform/opacity only) is Core Rule 2 in SKILL.md.

## Pipeline Cost Model

Style → Layout → Paint → Composite; the earlier a property enters, the more it costs per frame.

| You change | Pipeline from | Cost |
|---|---|---|
| `width`, `height`, `margin`, `top/left`, font-size | Layout | Worst — geometry of the whole subtree recomputes |
| `background`, `color`, `box-shadow`, `border-radius` | Paint | Moderate — pixels redraw, no geometry |
| `transform`, `opacity` | Composite | Cheapest — GPU repositions existing layers |

## Core Web Vitals (canonical numbers)

Google's thresholds, measured at the 75th percentile of page loads:

- CLS: good < 0.1, poor > 0.25. CSS owns this one almost entirely — reserved media space, font metrics, `scrollbar-gutter`.
- LCP: good ≤ 2.5s, poor > 4s. CSS levers: critical CSS inline, no `@import`, `font-display` on the hero font.
- INP: good ≤ 200ms, poor > 500ms. CSS lever: cheap style recalculation (containment, no `body:has()` invalidation storms).

## Compositing and will-change

- A composited layer holds GPU memory ≈ width × height × 4 bytes (RGBA): a 1000×1000px layer ≈ 4MB. `will-change` on every list item = layer explosion; scrolling gets worse, not better.
- Correct use: add `will-change: transform` just before animating (class or JS), remove after. Permanent `will-change` is only defensible on an element that animates constantly (a persistent FAB, a canvas).
- `transform: translateZ(0)` is the legacy spelling of the same hint — replace on sight.

## Layout Thrashing

- These reads force synchronous layout when styles are dirty: `offsetTop/Left/Width/Height`, `clientWidth/Height`, `scrollTop/Width`, `getBoundingClientRect()`, `getComputedStyle()` for geometry.
- The bug is interleaving: write-read-write-read in a loop = one full layout per iteration. Batch all reads, then all writes; schedule writes in `requestAnimationFrame`.
- Animating layout properties anyway? Use FLIP: measure First and Last rects, apply the inverted transform instantly, then transition the transform to identity — layout cost once, animation on the compositor.

## Containment

- `contain: layout` / `paint` on independent widgets: internal changes stop invalidating the page outside.
- `content-visibility: auto` skips rendering offscreen sections — the single biggest win on long pages. Pair with `contain-intrinsic-size: auto 500px`: your estimate prevents scrollbar jumping, `auto` remembers the real size once rendered.

## Fonts

- Body text: `font-display: swap` + metric overrides on the fallback `@font-face` (`size-adjust`, `ascent-override`, `descent-override`) so the swap causes no shift — tools generate these from the font files; this is the fix for font-driven CLS, not `optional`.
- `font-display: optional` for decorative fonts only — it may simply never show.
- Preload only the 1-2 WOFF2 files used above the fold (`<link rel="preload" as="font" crossorigin>`); every preload competes with the LCP image for bandwidth.
- WOFF2 only; other formats in 2025 are dead weight.

## Loading

- `@import` serializes: each level costs a full extra round trip after the parent sheet arrives. Always `<link>`; bundlers flatten it, raw CSS must never ship it.
- Inline critical CSS ceiling: the folk 14KB figure comes from TCP's initial congestion window (10 packets × ~1460B ≈ 14.6KB — one round trip). HTTP/3 blurs it, but it remains a sane budget for inlined `<head>` CSS.
- Unused CSS still parses on every load: DevTools Coverage tab before shipping a framework's full sheet.

## Verify Instead of Guessing

Every claim in this file is observable. Before and after any optimization, get the signal:

| Question | Where to look | What confirms it |
|---|---|---|
| Is my animation on the compositor? | Performance trace, or the Layers panel | No Layout/Paint entries during the animation; the element has its own layer |
| What is repainting? | Paint flashing overlay | Green flashes only over the animating element, not whole sections |
| Which element shifted? | Web Vitals overlay / Layout Shift entries | The named node plus its shift score |
| Is style recalculation the cost? | Performance trace, Recalculate Style entries | Duration and the "elements affected" count |
| Is the sheet oversized? | Coverage tab | Unused byte percentage on the routes that matter |
| Is a font blocking? | Network waterfall plus a screenshot filmstrip | Font request finishing after first text paint |

Optimizations applied without a before-number are indistinguishable from superstition — and several of them (`will-change`, extra layers, containment on the wrong node) make things worse.

## Selector Cost, the Reality

- Engines bucket rules by rightmost simple selector: a class or ID on the right makes the left side nearly free. Rewriting `.nav li a` for performance is folklore.
- What actually shows up in traces: `:has()` with broad subjects on mutating DOMs, universal `*` with expensive declarations, and mega-stylesheets where sheer rule count dominates.
- Rule: never restructure selectors without a Performance-panel trace showing Style/Recalculate as the cost. Otherwise it's readability debt for nothing.
