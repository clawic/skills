# Vectors and Icons

Icon work is where sloppiness compounds: a set built without a keyline drifts visually, and an icon built without export discipline arrives in code as a blurry, unrecolorable blob.

## The Keyline System

Build every icon inside the same square frame so consumers never have to compensate.

- Frame size: the `icon_grid` variable, default 24 px. Live area = `icon_grid − 4` (20 px at 24), leaving 2 px of padding on every side so icons never touch their bounding box.
- Keylines inside the live area: a square, a circle slightly larger than the square, and a horizontal and vertical rectangle. Optically, a circle must be a touch larger than a square to read as the same size — geometric equality reads as a smaller circle.
- Stroke weight consistent across the whole set (commonly 1.5-2 px at a 24 px grid). One icon at a different weight is visible in a row of twenty.
- Corner radius consistent: pick one value for outer corners and one for inner, and never mix.
- Optical centering beats geometric centering for asymmetric shapes. A play triangle centered by bounding box always looks left-heavy; nudge it right until it reads centered.

## Strokes

- Stroke alignment: center, inside, or outside. Only center exists in SVG — inside and outside get converted to outlined paths on export, which changes the geometry and kills the ability to change stroke width in code.
- Consequence: build icons you intend to ship as strokes with center alignment, or accept that they ship as filled paths.
- Stroke weight does not scale when you resize a frame by dragging. The `K` scale tool scales strokes, radii and text along with geometry — that is the difference between resizing an icon and scaling it.
- An open path with a non-center stroke and round caps is the classic "it looked fine in Figma" export defect.

## Boolean Operations and Flatten

- Union, Subtract, Intersect, Exclude stay live and editable; the source shapes remain inside the operation.
- Flatten collapses to a single path and is irreversible in practice. Flatten when a shape ships and will never be edited; never flatten an icon you may recolor or restyle.
- Keep editable originals on a hidden `_source` page. Cleanup passes destroy the only copy otherwise.
- Masks are heavier than boolean subtract and complicate export. Prefer a boolean where the result is the same shape.

## Pixel Alignment

- Odd stroke widths on whole-pixel coordinates land on half pixels and render soft. Either use even widths, or offset by half a pixel deliberately.
- Snap-to-pixel-grid handles positions but not the geometry inside a scaled component. An icon designed at 24 and used at 20 is resampled — build the set at the sizes it will be used at, or accept the softness.
- Frame bounds on whole pixels; a 23.5 px wide icon frame guarantees a blurry export at every scale.

## Building the Set

- One component per icon inside a shared page, named `Icon / Category / Name` so the picker groups them.
- Single path, single fill where possible. One fill bound to a color variable is what makes recoloring work; three hardcoded fills is what makes it not work.
- Consumers reach icons through an instance-swap property with the icon set as preferred values, not by dragging from the assets panel into a component.
- Do not import a thousand icons from a general library. Curate the fifty the product uses; every unused icon is picker noise and library weight.
- Duplicated icons under different names are the most common icon-library defect. Name by concept (`Icon / Action / Delete`), not by shape (`Trash`), so nobody adds `Bin` next month.

## Cleaning Imported SVGs

Imported vectors arrive with nested groups, transforms, and inline styles that break recoloring and inflate node count.

1. Ungroup until a single vector layer remains, then flatten transforms by resizing to the target frame with `K`.
2. Replace all fills with one, bound to a color variable.
3. Remove clip paths and masks the shape does not need.
4. Rename the layer to the icon name; layer names travel into exported SVG `id` attributes.
5. Check node count. An auto-traced logo with thousands of nodes belongs as a raster or a hand-rebuilt path; vector node count is the second-largest cost in a slow file, after original-resolution raster.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Icons look soft at 1x | Odd stroke on half-pixel, or non-integer frame bounds | Snap the frame to whole pixels; even stroke widths |
| Icon cannot be recolored in code | Multiple fills, or strokes outlined on export | One fill bound to a variable; center-aligned strokes |
| One icon looks bigger than the rest | Geometric rather than optical sizing | Build to the keyline set; circles slightly larger |
| Stroke thickened after resizing | Dragged instead of scaled | Use `K` to scale |
| Exported SVG has huge markup | Nested groups, transforms, clip paths | Flatten transforms, ungroup, remove clips |
| Play triangle looks off-center | Bounding-box centering | Nudge to optical center |
| Two icons for the same concept | Named by shape, not by concept | Rename by concept, deprecate the duplicate |
| Editable original lost after cleanup | Flattened without a copy | Keep a `_source` page |
| Icon frame sizes vary across the set | No shared keyline frame | Rebuild every icon inside the `icon_grid` frame |
