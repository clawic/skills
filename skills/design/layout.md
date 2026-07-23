# Layout — Grids, Alignment, Composition, Responsive

## Grid Defaults

- Web pages: 12-column grid, 24px gutters, content max-width 1140-1280px. 12 divides by 2, 3, 4, 6 — every common split works without remainder.
- Prose: single column, 560-600px (keeps 70-75ch at 16px). Never pour body text across a 12-column span wider than that.
- Dashboards: full-width, 24px outer margins, cards on the grid with equal gutters.
- Print/slides: margins first, then divide the live area. A document with generous margins and no grid beats a grid with cramped margins.

## Alignment

- Every element's left edge (right edge in RTL) sits on one of <= 3 vertical lines per view (Core Rule 6). Count them explicitly during review.
- Mixed alignment inside one group is the invisible killer: a centered heading over left-aligned body forces two reading entry points. Pick one per group.
- Baseline alignment across columns: text blocks side by side align on their first baseline, not their container tops — container-top alignment with different type sizes looks drunk.
- Numbers in columns right-align with tabular figures; text left-aligns; never center table columns (`ui-screens.md`).

## Composition (posters, heroes, covers)

- One focal point. Position it at the optical center — slightly above geometric center — or on a rule-of-thirds intersection for photographic layouts.
- Scan patterns are load-bearing: F-pattern for text-heavy pages (hierarchy hugs the left edge), Z-pattern for sparse layouts (logo top-left, action bottom-right). Place rank-1 and the CTA on the pattern, not against it.
- Direct attention with vectors: faces and arrows in imagery should point INTO the content, never off the page edge.
- Asymmetry is fine; imbalance is not. Balance mass (size x darkness x saturation), not element count — one dark photo equals a full column of text.

## Density Modes

- Comfortable (default): base 8 scale as in Core Rule 3.
- Dense (pro tools, tables, admin): `base_unit` 4 — 4px label-to-field, 12px field-to-field, 24px section-to-section; body may drop to 14px but never below 13px.
- Spacious (marketing, editorial): double section gaps (96px+), oversized display type; content per viewport drops, impact per element rises.
- Never mix modes in one view; a dense table inside a spacious page gets its own visually separated region.

## Responsive Behavior

- Common breakpoints: 640 / 768 / 1024 / 1280px. Break where the CONTENT breaks (measure exceeds 75ch, cards drop below usable width), not at device names.
- Columns collapse in rank order: rank-3 sidebars drop or fold first; rank-1 content and primary action stay above the fold at every width.
- Touch targets 44pt/48dp minimum on mobile; row height in dense tables can undercut this on desktop pointer devices only.
- Text does not scale linearly: body stays 16px at every width; display type shrinks from e.g. 39px to 31px (one scale step down) on mobile rather than proportionally.
- Test the awkward middle (768-1024px): two-column layouts with a squeezed main column fail here most often — go single column earlier rather than shipping a 40ch main column.

## Page Rhythm (long pages)

- Establish a vertical rhythm: consistent section padding (e.g. 96px marketing, 48px product) so scrolling has a beat.
- Alternate section treatments (background tint, image side) at most every other section; alternating every section reads as stripes.
- Anchor each section with one heading on the scale; equal-looking sections make the page feel endless (no progress markers).

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Full-width body text on desktop | 120ch+ measure, eye loses the return sweep | Cap prose at 560-600px regardless of page width |
| Breakpoints copied from a device list | Content breaks at its own widths | Break where measure or card width degrades |
| Centering a layout by padding both sides ad hoc | Edges drift off any shared line | Grid with max-width + auto margins |
| Equal vertical gaps everywhere | Grouping disappears, page feels both cramped and empty | Between-group >= 2x within-group, always |
| Hiding content on mobile to "simplify" | Rank-1 content missing for majority-mobile audiences | Collapse rank-3 first; reorder, don't remove rank-1 |
