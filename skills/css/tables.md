# Data Tables

Sizing, sticky headers, and making a table survive small screens without destroying its semantics.

## Sizing and the Two Layout Algorithms

- `table-layout: auto` (default) measures every cell before deciding column widths: content-driven, and increasingly slow as rows grow — the browser cannot lay out row 1 until it has seen row 10,000.
- `table-layout: fixed` + an explicit `width` uses only the first row (or `<col>` widths) and paints incrementally. Default for anything over roughly a hundred rows or any table that must not reflow as data loads.
- With fixed layout, set widths once via `<col>` elements or first-row cells; everything else is ignored, and long content needs `overflow-wrap: anywhere` or truncation.
- Numeric columns: `text-align: right` plus `font-variant-numeric: tabular-nums` so digits line up and do not jitter on update.
- Action columns: `white-space: nowrap; width: 1%` — the 1% trick shrink-wraps a column to its content under auto layout.

## Sticky Headers and Columns

```css
.table-scroll { overflow: auto; max-block-size: 70vh; }
table { border-collapse: separate; border-spacing: 0; }
thead th { position: sticky; top: 0; z-index: 2; background: var(--surface); }
tbody th[scope="row"] { position: sticky; left: 0; z-index: 1; background: var(--surface); }
thead th:first-child { z-index: 3; } /* the corner cell wins both */
```

- `border-collapse: collapse` and sticky do not mix: collapsed borders belong to the table, not the cell, so they scroll away and leave the header floating with no bottom edge. Use `separate` plus `box-shadow: inset 0 -1px var(--border)` on the header cells for the line.
- Sticky cells need an opaque `background`: row backgrounds paint below the cell, so a transparent sticky cell shows the scrolling content through it.
- Sticky works on `th`, `tr`, and `thead` in current engines; applying it to `th` is the widest-support choice and the one that survives older Safari.
- The scroll container must have a bounded size (`max-block-size` or a fixed height) or nothing ever scrolls, and nothing ever sticks.

## Small Screens

Two honest options, and one that lies:

1. **Horizontal scroll** — wrap in `overflow: auto` with `tabindex="0"` and `role="region"` + `aria-label` so keyboard users can scroll it. Keeps every relation intact; default choice.
2. **Stacked cards** — at a breakpoint, `display: block` on rows and `content: attr(data-label)` on cells. Readable, but it destroys the table role for screen readers unless you re-add ARIA roles, and every cell now needs a `data-label` in the markup. Choose it only when the table is a list of records people read one at a time.
3. **Hiding columns by breakpoint** — quietly deletes data. Acceptable only with a visible control that restores them.

- Add scroll affordances: an edge shadow via a background-attachment trick, or a persistent hint, so people know the table continues.
- Column priority beats font shrinking: below roughly 12px table text stops being usable, and 3px of padding is not a responsive strategy.

## Rows and States

- Zebra with filtering: `tbody tr:nth-child(odd of :not([hidden]))` — plain `:nth-child(odd)` double-stripes as soon as rows hide.
- Row hover on a table with sticky columns must set the background on the CELLS (`tr:hover > *`), because a sticky cell paints its own background over the row's.
- Selected rows: style with a `[aria-selected="true"]` hook, not a class only JS knows about, so state and semantics agree.
- Row-level links: a whole clickable row needs a real link in a cell, expanded with `::after { position: absolute; inset: 0 }` inside a `position: relative` row — never a click handler on `<tr>` alone.

## Structure and Semantics

- `<caption>` is the accessible name of the table; `caption-side: bottom` styles it without moving it in the DOM.
- `scope="col"` / `scope="row"` on header cells is what turns a grid of text into a navigable table; CSS cannot substitute for it.
- Do not `display: grid` a `<table>`: it drops the table role in several screen readers. If you need grid layout, use divs with explicit ARIA table roles, or keep the table and lay out around it.
- Long tables: `content-visibility: auto` plus `contain-intrinsic-size` on `<tbody>` row groups cuts rendering cost of offscreen rows; true virtualization is a JS concern.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Sticky header with `border-collapse: collapse` | Borders scroll away; header looks detached | `separate` + inset box-shadow |
| Transparent sticky cell | Content shows through while scrolling | Opaque background on the cell |
| `table-layout: auto` on huge tables | Full measure pass before first paint | `fixed` + `<col>` widths |
| Stacked-card responsive tables everywhere | Loses the table role and needs per-cell labels | Scroll container by default |
| Percentage widths on every column | They renormalize; total drifts from 100% | Set widths on `<col>` and let one column absorb the rest |
| Proportional digits in a metrics table | Columns visibly shift on each refresh | `tabular-nums` |
