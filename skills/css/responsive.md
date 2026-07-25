# Responsive Techniques

Fluid sizing math, viewport units, container queries. Accessibility thresholds (zoom, touch targets) live in SKILL.md — Accessibility Floor.

## Viewport Units

| Need | Unit | Why |
|---|---|---|
| Full-screen hero on mobile | `svh` | Excludes browser UI — content never hides behind the toolbar |
| App shell that must track the toolbar | `dvh` | Resizes live as UI retracts; causes reflow, keep the subtree cheap |
| Element sized against the page scroller inside a sub-scroller | `svh`/`dvh` still refer to the viewport | Viewport units never refer to a scroll container |
| Desktop-only layouts | `vh` | Stable there; the mobile toolbar problem doesn't exist |
| Default when unsure | `svh` | The only one that is both stable and never overlapped |

## Fluid Typography, Derived Not Guessed

Formula for `clamp(MIN, REMs + VWs, MAX)` between two viewports:

- slope = (maxSize − minSize) / (maxViewport − minViewport)
- vw term = slope × 100
- rem term = minSize − slope × minViewport (convert px→rem by dividing by `rem_base`, 16 by default)

Worked: 16px→24px across 400px→1280px viewports: slope = 8/880 = 0.00909 → `0.91vw`; rem term = 16 − 0.00909×400 = 12.36px → `0.77rem`. Result: `font-size: clamp(1rem, 0.77rem + 0.91vw, 1.5rem)`. Check both ends: at 400px → 12.36 + 3.64 = 16px; at 1280px → 12.36 + 11.64 = 24px.

- The rem term is non-negotiable: it is what makes zoom and user font-size scale the text (WCAG numbers: SKILL.md).
- Same formula works for fluid padding and gaps.

## Container Queries

- `container-type: inline-size` on the ancestor; `@container (min-width: 400px)` styles DESCENDANTS — a container can never style itself from its own query.
- The invisible side effect: inline-size containment means the container's width can no longer come from its content. If a shrink-wrapped component collapses to zero width after you add `container-type`, this is why — give the container a width source.
- `container-name` when containers nest: `@container card (min-width: 300px)` targets the named ancestor, not the nearest.
- Style queries — `@container style(--variant: featured)` — switch component variants off a custom property; Chromium-first, check support before relying on it.
- Division of labor: container queries for reusable components, media queries for page scaffolding (columns, nav) only.

## Container Query Units

- `cqi` / `cqb` (inline and block size of the query container), `cqmin` / `cqmax`. They resolve against the container, so a component sized in `cqi` adapts without knowing the page.
- Fluid type inside a component: use `cqi` in place of `vw` in the clamp formula above — same math, container-relative. Keep the rem term for zoom.
- Units resolve against the nearest ancestor with a `container-type`; with none, they fall back to the small viewport (`svi`), which looks correct for the wrong reason.

## Query Capability, Not Just Width

| Feature | Question it answers | Use |
|---|---|---|
| `(hover: hover)` | Can the primary input hover? | Gate hover-only affordances so they do not stick on touch |
| `(pointer: coarse)` | Is the primary pointer imprecise? | Raise target sizes and spacing (SKILL.md — Accessibility Floor) |
| `(any-pointer: fine)` | Is a mouse available at all? | Keep dense tools available on hybrid laptops |
| `(prefers-reduced-motion)` | Motion sensitivity | Wrap animation (SKILL.md Core Rule 2) |
| `(prefers-reduced-transparency)` | Blur and translucency sensitivity | Swap frosted panels for solid surfaces |
| `(prefers-color-scheme)` | Light or dark preference | Swap semantic tokens; pair with `color-scheme` so UA widgets follow |
| `(prefers-contrast: more)` | Wants stronger separation | Thicken borders, raise contrast |
| `(forced-colors: active)` | System palette override | Verify affordances survive: borders, focus rings, icon-only buttons |
| `(orientation: landscape)` | Wide-but-short | Phone landscape with a keyboard open is the real case |
| `(display-mode: standalone)` | Installed PWA | Reclaim the space a browser toolbar was taking |
| `(scripting: none)` | JS unavailable | Reveal no-JS fallbacks without a `<noscript>` block |

Width tells you the box; these tell you the human and the device. A hover menu behind `@media (min-width: 1024px)` still breaks on a 1024px touchscreen.

## Breakpoints

- Set breakpoints where the CONTENT breaks: start narrow, widen until the layout looks wrong, cut there. Device-width lists (768/1024) encode 2015 hardware, not your design.
- `em` vs `px` media queries: px queries follow page zoom but ignore the user's default font-size setting; em queries follow both. Content-driven breakpoints in em (`40em` = 640px at the 16px default) fire where the text actually needs them.
- Never mix `min-width` and `max-width` regimes in one file. Range syntax closes the boundary bug: `@media (400px <= width < 800px)` — with min/max pairs, exactly-800px matched both sides.

## Images

- Layout-shift prevention: `width`/`height` attributes on `<img>` (the browser derives aspect-ratio) or CSS `aspect-ratio`, plus `max-width: 100%; height: auto`.
- `srcset` + `sizes` = same picture, different resolutions (browser picks). `<picture>` = different crops per breakpoint (art direction). Using `srcset` for crops silently serves the wrong composition.
- `object-fit: cover` crops from the center; set `object-position` to protect the focal point (faces sit in the upper third — default center crop decapitates portraits).

## Device Reality Checks

- Emulators miss: live toolbar viewport changes, real touch-target ergonomics, notches. For edge-to-edge layouts: `viewport-fit=cover` + `padding: env(safe-area-inset-bottom)` on bottom bars.
- Landscape phone is its own case: ~400px height with a keyboard open — test forms there, not just portrait/desktop.
