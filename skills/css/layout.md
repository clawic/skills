# Layout Patterns

Debugging procedures and the layout mechanisms that break silently. Centering default and flex/grid sizing model live in SKILL.md.

## Layout Debugging Order

1. See real boxes, not assumed ones: DevTools flex/grid inspector, or temporarily `* { outline: 1px solid red }` (outline never affects layout — that is why it beats border for this).
2. Find the overflowing element before styling anything: in the console, walk `document.querySelectorAll('*')` for `el.scrollWidth > el.clientWidth`. Horizontal page scroll always has one concrete culprit.
3. Read Computed, not Styles: the authored value may be losing the cascade or being clamped by min/max constraints.
4. Only then write CSS — targeting the mechanism you identified, not the symptom.

## Sticky Failure Diagnosis

`position: sticky` fails silently. Check in this order:

1. Any ancestor between the element and the page with `overflow: hidden/auto/scroll`? That ancestor is now the scrollport; if it doesn't scroll, sticky "does nothing". Fix: `overflow: clip` on the ancestor — clip does not create a scroll container, so stickiness returns to the page scroller.
2. Is the sticky element as tall as its container? Then it has no room to travel. The container must be taller than the element.
3. Flex/grid parent? Default `align-items: stretch` causes case 2. Fix: `align-self: start` on the sticky child.
4. Missing inset: `top: 0` (or another inset) is required — sticky without an offset never engages.

Most developers only know step 4; steps 1-3 are the actual bugs.

## Height 100% and the Sticky Footer

- `height: 100%` resolves against the parent's DEFINED height — one unsized ancestor breaks the whole chain. Don't build chains.
- Sticky footer, the whole pattern: `body { min-height: 100svh; display: flex; flex-direction: column; }` + `footer { margin-top: auto; }`.
- App shell that must fill the screen: `min-height: 100dvh` on the shell only if live toolbar resize is acceptable (unit tradeoffs: `responsive.md`).

## Margin Collapse, the Actual Rules

- Only the block axis, only block layout. Flex, grid, inline-block, floats, and BFC roots never collapse.
- Parent-child bleed-through: a child's `margin-top` escapes the parent when nothing (border, padding, inline content) separates them — the classic "why did the whole container move down". Fix: `display: flow-root` on the parent, or switch to gap-based spacing.
- Adjacent siblings: the LARGER margin wins, they don't add. One negative: they sum (`24px + -8px = 16px`).
- Practical stance: use `gap` and single-direction margins (`margin-block-end` only); collapse then never fires.

## Overflow Semantics

- `hidden` vs `clip`: `hidden` creates a scroll container (JS can still scroll it, sticky descendants re-anchor to it); `clip` just clips — cheaper, keeps sticky working, allows `overflow-clip-margin`.
- One visible axis is impossible: if either axis is non-visible, `visible` on the other computes to `auto`. `overflow-x: clip; overflow-y: visible` silently becomes clip/auto — that's why the vertical scrollbar appears.
- Single-line truncation needs the trio: `overflow: hidden; white-space: nowrap; text-overflow: ellipsis` PLUS a constrained width (`min-width: 0` when in flex).
- Multi-line clamp: `display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 3; overflow: hidden` — prefixed but works in every modern engine.

## Animating to Height Auto

- Portable trick: wrapper `display: grid; grid-template-rows: 0fr; transition: grid-template-rows 0.3s`; open state sets `1fr`. The content child needs `min-height: 0; overflow: hidden`. Works everywhere grid does.
- Native path — `interpolate-size: allow-keywords` / `calc-size()` — is Chromium-only as of 2025; treat the grid trick as the default.

## Logical Properties

- If RTL or vertical writing modes are in scope, write `margin-inline`, `padding-block`, `inset-inline-start`, `inline-size` from day one — retrofitting is a full-file rewrite.
- Never mix physical and logical properties on the same box side; last-write-wins across the two systems is unreadable in review.
