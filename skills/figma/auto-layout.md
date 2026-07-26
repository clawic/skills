# Auto Layout — The Sizing Engine

Auto layout is a flex engine with a different vocabulary. Almost every "Figma is being weird" report is a sizing mode assigned to the wrong layer, and the fix is always found by resolving outside-in.

## Resolution Order

Figma sizes containers from the outside in. A child cannot know how much space to Fill until its parent knows its own width.

1. Set the outermost frame to Fixed (a page canvas) or Fill (a nested section).
2. Work down one level at a time; the first level where the size stops making sense is where the bug is.
3. Never start from the leaf. Fixing a button's width inside a broken card just moves the symptom.

Symptom of the circular case: a frame that will not respond to dragging at all — a Fill child inside a Hug parent on the same axis has no defined size, and Figma pins it to whatever width it had at the moment the modes were set.

## Clamps Replace Breakpoints

- `min-width` / `max-width` / `min-height` / `max-height` apply to Fill and Hug layers. Fill with a 320/1200 clamp inside a Fill parent centers and holds across the whole viewport range, with zero breakpoint frames.
- Hug plus `min-width` is the correct recipe for "a button that never gets narrower than its target size" — Hug alone collapses to the label.
- Max-width on the text container, not on the text node, is what produces a readable measure that survives a wide viewport.
- Clamps do not remove the need for real breakpoint frames when the *arrangement* changes (sidebar becomes a drawer). They remove the frames that only changed a width.

## Wrap

- Available on horizontal layouts. Once on, gap splits into horizontal and vertical values and alignment applies per row.
- Chips, tag rows, filter bars, and card grids that must reflow: wrap on the parent, Hug on the children.
- Fill children inside a wrapping row each take a full row — that is the engine working as designed, not a bug. Use Hug children with clamps for a reflowing grid.
- Wrap before nesting manual rows. Manual row-per-line structures break the instant content count changes.

## Grid Layout

The grid layout mode arranges children in rows and columns inside a single auto layout frame, with per-child span. It replaces the nested "row of columns of rows" structure for dashboards, bento layouts, and any arrangement with alignment across both axes.

- Reach for grid when children must align across rows *and* columns; stack layouts only guarantee one axis.
- A child that spans two columns stays one layer — the old solution was a wrapper frame per span, which broke every reorder.
- Grid is not layout grids. Layout grids are a visual overlay and a constraint target in fixed manual-layout frames; they position nothing inside an auto layout frame.

## Absolute Position

- Pulls a child out of flow while keeping it parented: badges on avatars, close buttons on cards, FABs, tooltips that belong to the component.
- Constraints become active on that child (and only that child), so it can pin to a corner and scale with the parent.
- An absolutely-positioned child does not contribute to a Hug parent's size. That is why the notification badge hanging off the top-right corner gets clipped: the parent hugged the content it could see. Fix by adding padding to the parent or turning off `Clip content`.
- Prefer absolute position over a separate overlay frame whenever the element belongs to the component's anatomy — it travels with every instance.

## Spacing, Padding, and Alignment

- Unlink the padding chain for per-side values. Asymmetric padding is first-class; faking it with nested frames doubles the layer count for nothing.
- Negative gap plus canvas stacking order (`Last on top` / `First on top`) produces overlapping avatar stacks without absolute positioning.
- Baseline alignment on a horizontal row aligns text by its baseline rather than its bounding box — the fix for an icon-plus-label row where the label sits one pixel high.
- `Clip content` is off by default on new frames: overflow spills silently over siblings and only shows up when someone screenshots the wrong thing. Turn it on for any scrollable or masked region.
- The stroke-inclusion toggle decides whether strokes count toward layout bounds. A 1px border that shifts everything by 2px total is this setting.

## Nesting Discipline

- Every auto layout frame is a flex container. Six levels of nesting to render a card means five of them are doing nothing that padding and gap could not.
- Spacer frames are legitimate exactly once: a Fill spacer inside a Packed row where one item must sit flush right and `Space between` cannot be used (because there are three items and only one gap should grow). Everywhere else, `Space between` is the answer.
- `Shift + A` wraps the selection in a new auto layout frame. Pressing it twice nests a second frame — the most common accidental extra layer in any file.
- Collapse pass before handoff: select the card, and for each wrapper ask what property it carries. Wrappers with no padding, no gap, no fill, and no name are deletable.

## The Drag Audit

Static canvas lies about responsive behavior. The audit is 30 seconds and catches most handoff defects.

1. Grab the right edge of the top frame and sweep from the minimum supported width to the maximum.
2. Watch for: text clipping, elements overlapping, a fixed-width child forcing a horizontal overflow, an image that stretches instead of cropping.
3. Paste the longest realistic string into every text node (a 40-character surname, a 7-digit count, a two-line product title) and sweep again.
4. Anything that breaks is wrong even if the design at the default width is perfect.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Frame ignores every drag | Fill inside Hug on the same axis | Set the outermost container to Fill or Fixed, resolve inward |
| Child overflows the parent instead of shrinking | Child is Fixed, or has a `min-width` above the available space | Fill the child, or lower the clamp |
| Badge is cut off at the corner | Absolutely-positioned child ignored by a Hug parent | Add parent padding, or turn off `Clip content` |
| Single child sits left when it should sit right | `Space between` with one child | `Packed` plus a Fill spacer, or align the child |
| Constraints panel has no effect | Inside an auto layout frame and the child is in flow | Change the sizing modes, or make the child absolute |
| Content spills invisibly over the next section | `Clip content` off | Turn it on for masked and scrollable regions |
| Row items misaligned by a pixel or two | Bounding-box alignment on mixed icon and text heights | Baseline alignment on the row |
| Everything shifted after adding a border | Stroke counted in layout bounds | Toggle stroke inclusion, or move the border to the parent |
| Grid children reorder unpredictably | Spans defined by wrapper frames instead of grid spans | Rebuild as a grid layout with per-child span |
