---
name: css
slug: css
version: 1.0.3
description: Writes and debugs modern CSS - stacking contexts, flexbox/grid, responsive layout, selectors, render performance. Use for layout bugs, z-index issues, CLS, fluid typography, or modernizing stylesheets.
homepage: https://clawic.com/skills/css
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🎨
    os:
    - linux
    - darwin
    - win32
    displayName: CSS
---

## When To Use

- Debugging layout: z-index that won't apply, overflow, dead `height: 100%`, broken `position: sticky`
- Building components: flexbox/grid patterns, centering, responsive behavior without media-query sprawl
- Production hardening: layout shift, animation jank, font loading, the accessibility floor
- Replacing JS or preprocessor hacks with native CSS (`:has()`, `@layer`, container queries, scroll snap)
- Not for visual design decisions (palettes, spacing scales, typography choice) — this skill covers mechanics, not taste

## Quick Reference

| Situation | Play |
|---|---|
| z-index ignored despite a huge value | Stacking Contexts below — find the context root; never just bump the number |
| Flex item overflows / text won't truncate | `min-width: 0` on the flex child (default min-width is min-content) |
| Breaks with real content, sticky dead, footer floats, margin leaks | `layout.md` |
| Component must adapt to its container; fluid type; mobile viewport bugs | `responsive.md` |
| Specificity fight, `@layer`, `:has()`, custom-property gotchas | `selectors.md` |
| Jank, layout shift, slow paint, font flash | `performance.md` |
| Anything else CSS | Core Rules below, then the closest file above |

## Core Rules

1. Diagnose before adding CSS: reproduce, isolate in DevTools, name the mechanism (stacking context, flex sizing algorithm, margin collapse). A property added without a named mechanism is the next bug.
2. Animate only `transform` and `opacity` — the only common properties that skip layout and paint. Frame budget = 1000ms / 60fps ≈ 16.7ms for style, paint, and your JS combined; one layout-triggering animation spends it alone.
3. One centering default: parent `display: grid; place-content: center`. Escape hatch: `position: absolute; inset: 0; margin: auto` when the child must overlay (needs a resolvable size, e.g. `width: fit-content`).
4. Never bare viewport units for text. `font-size: clamp(1rem, 0.77rem + 0.91vw, 1.5rem)` — the rem term is what keeps browser zoom and user font-size working; pure-vw text fails WCAG 1.4.4 (resize to 200%). Derivation of the numbers: `responsive.md`.
5. Size intrinsically first (`min()`, `clamp()`, `fit-content`, `auto-fit` grids), media queries second, container queries when one component lives at different widths.
6. `!important` in component code is a debt marker. Order wars belong in `@layer` — unlayered author styles beat all layered ones regardless of specificity (`selectors.md`).
7. Ship only after hostile-content testing: longest word (URL, German compound), empty state, missing image, 200% zoom, `prefers-reduced-motion`.

## Stacking Contexts

The single most common CSS debugging failure: raising z-index on an element trapped inside a context.

- Context creators (memorize): positioned element with z-index, flex/grid child with z-index, `opacity < 1`, `transform`, `filter`, `backdrop-filter`, `will-change`, `contain: layout` or `paint`, `position: fixed`/`sticky`, `isolation: isolate`.
- Inside a context, z-index competes only among siblings of that context. A child's `z-index: 9999` never escapes its parent's `z-index: 1`.
- Debug procedure, in order: (1) walk up from the losing element to its first context-creating ancestor; (2) same for the winning element; (3) compare those two ancestors — that comparison decides the paint order; (4) fix z-index there, or delete the accidental trigger (usually a leftover `transform` or `opacity` from an animation).
- `isolation: isolate` creates a context with zero visual side effects — use it to cap a component's internal z-index so it can't leak out.
- `transform`, `filter`, and `will-change` also make the element the containing block for `position: fixed` descendants — the fixed element silently behaves as absolute. Same walk-up diagnosis.

## Flexbox and Grid Mental Model

- `flex: 1` = `1 1 0%`: ALL space divided equally. `flex: auto` = `1 1 auto`: only leftover space divided, so larger content keeps a larger track. Choose per intent; equal columns need basis 0.
- Flex children default to `min-width: min-content` — the root cause of both overflow and un-truncatable text. Release with `min-width: 0` (or `overflow: hidden`). Column direction: same story with `min-height`.
- `1fr` means `minmax(auto, 1fr)`: the track refuses to shrink below its content. `grid-template-columns: 1fr 1fr` is NOT 50/50 with unequal content — write `minmax(0, 1fr)` for true halves.
- `auto-fit` collapses empty tracks (remaining cards stretch); `auto-fill` keeps them (cards hold max width). Card grid default: `repeat(auto-fit, minmax(min(250px, 100%), 1fr))` — the inner `min()` prevents overflow on viewports under 250px.
- `gap` never collapses; margins collapse (vertical, block layout only, including parent-child bleed-through). Prefer gap and treat margin collapse as legacy behavior to route around (`layout.md`).
- `margin: auto` on a flex/grid child absorbs free space: `margin-inline-start: auto` on the last nav item is the entire "push right" pattern.

## Modern CSS Worth Using

Compatibility floor: everything here works in all evergreen engines since end of 2023 unless marked.

- `:has()` — parent and previous-sibling selection; kills a whole class of state-mirroring JS (`selectors.md` for patterns and cost).
- `@starting-style` + `transition-behavior: allow-discrete` — transition from `display: none`; replaces enter-animation JS (all engines since mid-2024).
- `text-wrap: balance` on headings — engines skip long blocks (Chromium caps at 6 lines), so it is safe to apply to all headings.
- `scrollbar-gutter: stable` on scroll containers — reserves the gutter, no shift when the scrollbar appears.
- `overscroll-behavior: contain` on modals and drawers — stops scroll chaining into the page.
- `scroll-snap-type` + `scroll-snap-align` — carousels without JS.
- `aspect-ratio` — reserve media space before load (layout-shift numbers: `performance.md`).
- `accent-color` — form controls on brand without rebuilding them.

## Accessibility Floor

Canonical home for these numbers — other files point here.

- Contrast (WCAG 2.2 AA): 4.5:1 body text; 3:1 for large text (≥24px, or ≥18.66px bold) and for UI components and focus indicators (1.4.3, 1.4.11).
- Touch targets: ≥24×24 CSS px is the AA minimum (2.5.8); 44×44 matches Apple HIG and AAA — use 44 for primary mobile actions.
- Text survives 200% zoom (1.4.4): rem-based sizes plus the clamp rule (Core Rule 4).
- Motion is opt-in: wrap animation in `@media (prefers-reduced-motion: no-preference)` rather than overriding after the fact.
- Style `:focus-visible`; never `outline: none` without a replacement in the same rule.
- `@media (forced-colors: active)`: system colors replace yours — check borders and focus still exist there.
- Dark mode: `@media (prefers-color-scheme: dark)` plus `color-scheme: light dark` so form controls and scrollbars follow.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Bumping z-index to 9999 | Element is inside a stacking context; only the context root competes outside | Walk-up procedure (→ Stacking Contexts) |
| Animating height/top/left/margin | Layout runs every frame and blows the 16.7ms budget (Core Rule 2) | `transform`; for height-to-auto, the grid-rows trick (→ layout.md) |
| `overflow: hidden` to kill a stray scrollbar | Hides the symptom and creates a scroll container: breaks sticky descendants, clips shadows and focus rings | Find the overflowing element first; `overflow: clip` if clipping is truly intended |
| `var(--x, fallback)` as a safety net | A declared-but-invalid value skips the fallback ("invalid at computed-value time") | `@property` with `initial-value` (→ selectors.md) |
| Global `will-change` or `translateZ(0)` "GPU hints" | Every layer holds GPU memory; hundreds of layers slow compositing | `will-change` only on elements actually animating, only while animating (→ performance.md) |
| `100vh` full-screen sections | Mobile browser UI overlaps the bottom of the section | `100svh`; `dvh` only when live resize is acceptable (→ responsive.md) |
| `!important` to win a specificity fight | Escalation is one-way; the next override needs another `!important` | `@layer` ordering (→ selectors.md) |
| `:empty` for empty states | Whitespace text nodes count as content in most engines | Control the markup, or a class set by the renderer |

## Where Experts Disagree

- Selector performance: the old guard writes selectors for right-to-left matching cost; modern engines bucket by rightmost simple selector, making it negligible. Boundary: only act on a DevTools trace showing Style/Recalculate cost — usually `:has()` or universal selectors on large, frequently-mutating DOMs (`performance.md`).
- Utility-first vs handwritten CSS: utilities win on team consistency and dead-code elimination; handwritten wins for animation-heavy and design-led work. Boundary: follow whichever the codebase already uses; never mix systems inside one component.
- CSS-in-JS: colocation and typed themes vs runtime cost. Boundary: server-rendered, performance-critical pages want zero-runtime extraction (or plain CSS + `@layer`); internal dashboards can afford runtime styling.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `html` - semantic structure and document-level concerns the CSS hooks into
- `frontend` - component architecture, frameworks, and build tooling around the styles
- `animations` - motion design and choreography beyond single-property transitions
- `accessibility-audit` - full WCAG review beyond the CSS floor here
- `design-system` - tokens, theming, and scaling styles across a product

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/css.
