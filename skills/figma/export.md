# Export — Assets, Densities, and Handing Off Files

Exporting is a contract with the build system: the wrong density, format, or naming turns into an engineer's manual conversion step every sprint.

## Export Settings

- Export settings attach to a layer, frame, section, or slice, and persist. Set them once on the icon component and every instance inherits the intent.
- Multiple rows per layer produce several files at once: `1x PNG`, `2x PNG`, `SVG`. This is how one selection satisfies three platforms.
- The suffix field is what names the file: `@2x`, `-dark`, `_android`. Slashes in the layer name create folders inside the exported zip, which is how a whole icon set arrives pre-organized.
- `Export contents only` (on frames with a fill or stroke) exports the children without the frame's own background — the difference between an icon and an icon on a white square.
- Slices export an arbitrary region regardless of layer bounds. Useful for pulling a fragment out of a dense composition; a liability when someone moves the artwork underneath.

## Densities by Platform

| Platform | Scales to export | Naming convention |
|---|---|---|
| Web (raster) | 1x, 2x | `image.png`, `image@2x.png` |
| Web (vector) | SVG | one file, no scaling |
| iOS | 1x, 2x, 3x | `image.png`, `image@2x.png`, `image@3x.png` |
| Android | mdpi 1x, hdpi 1.5x, xhdpi 2x, xxhdpi 3x, xxxhdpi 4x | one folder per density bucket |

- Design at 1x logical size and export up. Designing at 2x and dividing produces half-pixel values throughout the file.
- Android's 1.5x bucket is why odd base dimensions hurt: a 25 px asset becomes 37.5 px at hdpi. Keep base sizes even, and divisible by 4 where possible.
- Prefer vector for anything vector: SVG on web, PDF or a vector drawable pipeline on native. Density buckets only exist for raster.

## Format Choice

| Format | Use for | Watch for |
|---|---|---|
| SVG | Icons, logos, simple illustration | Text nodes, transforms, and non-center strokes need cleaning first |
| PNG | Raster with transparency, screenshots | File size — a full-bleed photo as PNG is many times a JPG |
| JPG | Photography with no transparency | Banding on gradients; no alpha |
| PDF | Print, multi-page decks, some native pipelines | Fonts and color profile behavior differ from screen |

Figma does not export modern raster formats like WebP or AVIF directly; that conversion belongs in the build pipeline, so hand off PNG or JPG masters at the highest needed density and let the pipeline generate the rest.

## SVG Options

- **Outline text** converts text to paths: guarantees the render, destroys selectability, searchability, and any chance of the engineer restyling it. Off for anything with real copy, on only for a wordmark whose font is not licensed for the web.
- **Include `id` attribute** carries Figma layer names into the SVG. Useful for animation and targeting, noisy otherwise — and it exposes internal naming, so review names before enabling it on anything public.
- **Simplify stroke** converts strokes to fills where Figma cannot express them otherwise. It silently removes the ability to set stroke width in CSS.
- Run exported SVGs through an optimizer in the build; hand-tuning markup in Figma is not the right layer for it.

## Image Handling

- Figma keeps the original bitmap regardless of how small the layer is on canvas. A 4000 px photo cropped into a 200 px avatar still carries the full original in the file and in every export derived from it.
- Downscale before placing, or crop-and-flatten a copy, when the original will never be re-cropped. This is simultaneously the export fix and the file-weight fix.
- Transparent background: turn off the frame's fill. Exporting a frame with a white fill and then keying it out downstream is a lossy round trip.
- Color profile matters for print and for wide-gamut displays; verify what the destination expects rather than assuming sRGB throughout.

## Who Exports

- One-off assets: the designer exports and attaches them to the ticket.
- A whole icon set or a recurring asset pipeline: the engineer pulls them, either through a plugin or through the image-render endpoint of the REST API. Manual export of a set that changes weekly is a standing tax.
- Either way, the asset names are the contract. Agree the naming convention before the first export, because renaming later breaks every reference in code.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Exported icon has a white square behind it | Frame fill exported with the contents | `Export contents only`, or remove the fill |
| Assets blurry on high-density screens | Only 1x exported | Add 2x and 3x rows, or ship SVG |
| Android assets land at fractional pixels | Odd base dimensions against the 1.5x bucket | Use even base sizes, divisible by 4 |
| SVG text renders in the wrong font | Text left as `<text>` with an unavailable font | Outline the text, or ship the font |
| SVG stroke width cannot be changed in CSS | Strokes outlined by simplify-stroke or non-center alignment | Center-align strokes; disable simplify |
| Export file sizes are enormous | Original bitmaps retained behind small crops | Downscale before placing |
| Exported filenames do not match code | Suffixes and slashes not set on the layer | Set the suffix and slash-name the layer |
| Sliced export changed after someone moved a layer | Slice region is independent of layer bounds | Export from the frame or component instead |
