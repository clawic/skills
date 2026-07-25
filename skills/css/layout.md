# Layout Patterns

Page scaffolding recipes and the layout mechanisms that break silently. Centering default and the flex/grid sizing model live in SKILL.md; symptom-first triage lives in `debugging.md`.

## Page Scaffolding Recipes

- **Content grid with full-bleed escape.** One grid, three named tracks, no negative margins and no `100vw` scrollbar bug:

```css
.page {
  display: grid;
  grid-template-columns:
    [full-start] minmax(1rem, 1fr)
    [content-start] min(65ch, 100% - 2rem) [content-end]
    minmax(1rem, 1fr) [full-end];
}
.page > * { grid-column: content; }
.page > .bleed { grid-column: full; }
```

- **Sidebar that collapses without a media query.** `grid-template-columns: repeat(auto-fit, minmax(min(20rem, 100%), 1fr))` for equal panes, or `flex-basis: 20rem; flex-grow: 1` on both children with `flex-wrap: wrap` when the sidebar should wrap under the main column at its own threshold.
- **Header / content / footer shell.** `body { min-height: 100svh; display: grid; grid-template-rows: auto 1fr auto }` — `1fr` on the middle row is what pins the footer down without `margin-top: auto` chains.
- **Overlapping hero layers.** Place both children in the same grid area (`grid-area: 1 / 1`) instead of absolute positioning: the tallest child still sizes the container.
- **Masonry.** No stable native solution yet (the spec is contested). Use CSS columns when reading order may go top-to-bottom per column, JS when it must not.

## Positioning Fundamentals

- `position: absolute` resolves against the nearest ancestor with a `position` other than `static` — or, failing that, the initial containing block (the viewport-sized box at the document root), which is why a stray absolute element flies to the page corner.
- `transform`, `filter`, `backdrop-filter`, `perspective`, `will-change` on those, and `container-type` also create containing blocks for fixed descendants (SKILL.md — Stacking Contexts).
- `inset: 0` plus `margin: auto` centers an absolute child with a resolvable size; `inset: 0` alone stretches it to fill.
- An absolutely positioned child of a grid container positions against the grid AREA when the parent is `position: relative` and the child has a `grid-area` — the cleanest way to pin a badge inside a cell.
- `position: static` is not "no position": it is what makes an element transparent to absolute descendants. Removing `position: relative` from a wrapper reparents every absolute child inside it.

## Intrinsic Sizing Keywords

| Keyword | Means | Use |
|---|---|---|
| `min-content` | Widest unbreakable word | Labels that must never wrap mid-word |
| `max-content` | Full width on one line | Chips and tabs that size to their text (dangerous in narrow parents) |
| `fit-content` | `min(max-content, available)` | Shrink-wrap that still respects the container |
| `fit-content(20rem)` | Same, capped | Grid tracks that grow to content up to a limit |
| `stretch` | Fill the containing block | Replacing `width: 100%` where padding would overflow |

`width: fit-content` plus `margin-inline: auto` is the shrink-wrap-and-center idiom that needs no flex parent.

## Subgrid and Card Alignment

- Cards in a grid whose titles, bodies, and footers must align across columns: `grid-template-rows: subgrid` on the card with `grid-row: span 3`, so each card's internal rows join the parent's track sizing. All engines since late 2023.
- Without subgrid, the fallback is a fixed row template on the parent plus `min-height` on the card sections — it works and it lies whenever content lengths differ.
- `align-content` vs `align-items`: `content` moves the whole track set inside a taller container; `items` moves each item inside its track. Wrong one chosen is the "my centering only works sometimes" bug.

## Display Contents and Table Displays

- `display: contents` removes the box but keeps the children in the parent's layout — the way to let a semantic wrapper's children participate in a grid. Historically it removed the element's semantics in some screen readers; it is fixed in current engines, but avoid it on elements with meaning (`ul`, `table`, form groups) when older assistive tech is in scope.
- `display: table-cell` still beats flexbox for one thing: vertical centering of unknown content inside a fixed-height box in legacy targets. Everywhere else, grid.

## Block Formatting Contexts

A BFC contains floats, stops margin collapse through its boundary, and stops overlapping a sibling float. Created by: `display: flow-root` (the one with no side effects), `overflow` other than `visible`, floats themselves, absolutely positioned elements, flex/grid ITEMS, `contain: layout`, and table cells. When a container refuses to wrap around its floated child, `display: flow-root` is the answer — not a clearfix pseudo-element.

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
- App shell that must fill the screen: `min-height: 100dvh` on the shell only if live toolbar resize is acceptable; `100svh` is the stable default.

## Margin Collapse, the Actual Rules

- Only the block axis, only block layout. Flex, grid, inline-block, floats, and BFC roots never collapse.
- Parent-child bleed-through: a child's `margin-top` escapes the parent when nothing (border, padding, inline content) separates them — the classic "why did the whole container move down". Fix: `display: flow-root` on the parent, or switch to gap-based spacing.
- Adjacent siblings: the LARGER margin wins, they don't add. One negative: they sum (`24px + -8px = 16px`).
- Practical stance: use `gap` and single-direction margins (`margin-block-end` only); collapse then never fires.

## Overflow Semantics

- `hidden` vs `clip`: `hidden` creates a scroll container (JS can still scroll it, sticky descendants re-anchor to it); `clip` just clips — cheaper, keeps sticky working, allows `overflow-clip-margin`.
- Mixed axes, the actual computed-value rule: `visible` paired with `scroll`/`auto`/`hidden` computes to `auto` (so `overflow-x: hidden; overflow-y: visible` silently becomes hidden/auto and grows a vertical scrollbar), and `clip` paired with those computes to `hidden`. `clip` + `visible` is the one legal mixed pair and stays as authored in all three engines.
- That pair is the fix for a stray horizontal scrollbar you cannot remove at the source: `html { overflow-x: clip }` (Safari 16+) cuts the overflow without creating a scroll container, so `position: sticky` descendants keep sticking — which `overflow-x: hidden` on `html` or `body` destroys.
- `overflow-clip-margin: 8px` lets a clipped box keep a halo of overflow — enough room for a focus ring or a shadow without reopening the whole overflow.
- Text truncation (ellipsis, line clamp) is a typography concern, but its layout half belongs here: both need a width source, which in flex means `min-width: 0` on the truncating child.

## Animating to Height Auto

- Portable trick: wrapper `display: grid; grid-template-rows: 0fr; transition: grid-template-rows 0.3s`; open state sets `1fr`. The content child needs `min-height: 0; overflow: hidden`. Works everywhere grid does.
- Native path — `interpolate-size: allow-keywords` / `calc-size()` — is Chromium-only as of 2025; treat the grid trick as the default.

## Aspect-Ratio Boxes

- `aspect-ratio: 16 / 9` sizes the missing dimension from the given one; it is ignored when both are set, and it is a suggestion once content overflows (add `min-height: 0` or `overflow: hidden` to enforce it).
- Media: `img { aspect-ratio: attr(width) / attr(height) }` is not needed — supplying the `width`/`height` attributes already gives the browser the ratio, which is the layout-shift fix.
- Square avatars and video thumbs: `aspect-ratio` plus `object-fit: cover` beats padding-top percentage hacks, which required an absolutely positioned child.
- In a grid track sized by `1fr`, aspect-ratio fights the track: the track wins. Give the item a definite inline size or let the ratio drive the row with `grid-auto-rows: auto`.
