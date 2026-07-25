# Shadows, Filters, Masks, and Transforms

Visual effects and the geometry that drives them: transforms, 3D, shadows, filters, clipping, blend modes.

## Transforms

- Use the individual properties: `translate`, `rotate`, `scale`. They compose in a fixed order (translate → rotate → scale) and animate independently, which ends the classic collision where a hover `transform: scale()` wipes out a layout `transform: translateY()`.
- In the `transform` shorthand, order is application order: `translateX(10px) rotate(45deg)` moves then rotates; `rotate(45deg) translateX(10px)` moves along the rotated axis. Neither is wrong; both are common bug reports.
- `transform-origin` defaults to the box center; for a menu that grows from its trigger, set it to the corner the panel is anchored at.
- Transforms do not affect layout: siblings do not move, and the element still occupies its untransformed box. Scaling a card in a grid overlaps its neighbors rather than pushing them.
- A transformed ancestor becomes the containing block for `position: fixed` descendants and creates a stacking context (SKILL.md — Stacking Contexts).
- Percentage translations are relative to the element's OWN size — `translateX(-50%)` is the centering idiom precisely for that reason.
- SVG: `transform-box: fill-box` makes `transform-origin: center` mean the shape's center instead of the SVG viewport's.

## 3D

- Minimum viable card flip: `perspective` on the parent (600-1200px reads natural for card-sized elements; smaller values exaggerate), `transform-style: preserve-3d` on the rotating element, `backface-visibility: hidden` on both faces.
- `preserve-3d` is destroyed by `overflow: hidden`, `filter`, `opacity < 1`, and `clip-path` on the same element — the "my flip went flat" bug is always one of those on the wrong node.
- Perspective on the parent (`perspective: 800px`) shares a vanishing point across children; `transform: perspective(800px)` per child gives each its own and looks wrong in a grid.

## Shadows

- `box-shadow` follows the border box (and its radius); `filter: drop-shadow()` follows the alpha channel — the only way to shadow a transparent PNG, an SVG icon, or a `clip-path`ed shape.
- Depth reads better as two or three stacked shadows with different blur radii and low alpha than as one big soft shadow: a tight, near-opaque shadow for the contact edge plus a wide, faint one for ambient light.
- Shadow color should be a dark tint of the surface hue, not neutral black at high alpha — black shadows on colored surfaces read grey and dirty.
- `inset` shadows draw inside the padding box: the trick behind hairlines that respect `border-radius` and behind sticky table header borders.
- Animating `box-shadow` repaints; animate the opacity of a pseudo-element that carries the shadow instead.
- In dark mode, shadows are nearly invisible — express elevation with surface lightness.

## Filters and Backdrop Filters

- `filter` applies in order and each function costs a pass: `filter: saturate(1.2) blur(2px)` is two passes over the element's pixels.
- Any non-`none` `filter` creates a stacking context AND a containing block for fixed descendants — the accidental cause of half the "my sticky/fixed element broke" reports.
- `backdrop-filter: blur(12px)` needs the element itself to be translucent, or there is no backdrop to see. Keep the blurred area small: it samples and blurs everything behind it every frame it changes.
- Frosted panels need a fallback: `@supports not (backdrop-filter: blur(1px)) { .panel { background: var(--surface) } }`, because an unsupported blur leaves unreadable text over content.
- `filter: blur()` on a container blurs its children's text into unreadability at small radii — blur the background layer, not the content wrapper.

## Clipping and Masking

- `clip-path: inset(0 round 12px)` clips with rounded corners and, unlike `overflow: hidden`, cooperates with transforms (Safari has long-standing jitter with rounded overflow plus transforms).
- Animating `clip-path` between polygons requires the SAME number of vertices in both states; add redundant points to match.
- `mask-image: linear-gradient(to bottom, black, transparent)` is the portable fade-out edge for scroll containers and long text; `mask-composite` combines multiple masks.
- Masks apply to the alpha channel by default (`mask-mode`), so a black-to-transparent gradient is a visibility ramp, not a color.
- `clip-path` and `mask` both cut off shadows, focus rings, and outlines that would paint outside the shape — decorate inside, or wrap.
- `shape-outside` wraps text around a shape; unlike `clip-path` it changes layout, not painting.

## Blend Modes

- `mix-blend-mode` blends the element with everything painted below it in its stacking context — including the page background. Wrap the group in `isolation: isolate` so the blend stays local; without it, a `multiply` overlay tints the whole page it happens to sit on.
- `background-blend-mode` blends an element's own background layers (image + gradient) with no isolation concerns.
- `mix-blend-mode: difference` on a fixed cursor or heading is the standard "text inverts over any background" trick — verify contrast at both extremes; it can land on middle grey where nothing is readable.
- Blend modes force the element onto its own composited layer: a handful is fine, a list of hundreds is not.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `transform: scale()` overwriting a positioning transform | Shorthand replaces the whole value | Individual `scale`/`translate` properties |
| `filter` on a wrapper with `position: fixed` children | Wrapper becomes the containing block; fixed turns absolute | Move the filter to a leaf element |
| `backdrop-filter` on an opaque panel | No visible effect, full cost | Translucent background required |
| One 40px-blur shadow for elevation | Reads as a grey smudge and repaints expensively | Layered shadows, low alpha |
| `overflow: hidden` for rounded corners over a transform | Sub-pixel jitter and clipped focus rings, notably in Safari | `clip-path: inset(0 round …)` |
| `mix-blend-mode` without an isolated parent | Blends against the entire page | `isolation: isolate` on the wrapper |
| `preserve-3d` plus `overflow: hidden` on the same node | 3D context is flattened with no error | Split into two elements |
