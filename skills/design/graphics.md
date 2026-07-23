# Graphics — Posters, Social Posts, Thumbnails, Banners

One-glance artifacts: seen small, in a feed, for about a second. Different physics from pages — hierarchy compresses to ONE message plus one visual.

## The One-Glance Rules

- Single message. A poster with two messages delivers zero; move everything secondary to small print or off the artifact.
- Readable at 10% zoom: thumbnail the design; if the message survives, it works in the feed. This is the squint test with real stakes.
- Contrast over palette: one-glance artifacts earn attention through a strong lightness contrast (dark/light blocking), not through color count.
- Type IS the design for most graphics: one huge line (the message) + one small line (the detail). Display type at poster scale wants tightened tracking (-2%) and manual line breaks at phrase boundaries — never let a headline break mid-phrase.

## Poster Hierarchy

- Rank-1 at visual center or upper third, sized to dominate: the headline occupies roughly a third of the composition's area.
- The reading gravity: big message → supporting visual → details (date/place/URL) in ONE small cluster near the bottom edge. Scattered detail lines are the amateur tell on posters.
- Viewing distance sets the floor: street posters need the message legible at 3-5m — when in doubt, double the size you first set.
- Rule-of-thirds placement for photographic posters; scrim under any text on photo until contrast passes (`palettes.md`).

## Common Canvas Sizes

Platform-current as of 2026 — verify before pixel-perfect work; aspect ratio matters more than exact pixels:

| Use | Size | Note |
|---|---|---|
| Open Graph / link preview | 1200x630 | Text in the center ~80%; edges crop on some surfaces |
| Square feed post | 1080x1080 | The safe cross-platform default |
| Portrait feed post | 1080x1350 | More screen area in feeds than square |
| Story / Reel / vertical | 1080x1920 | Keep text inside the middle ~70% vertically; UI chrome overlays top and bottom |
| YouTube thumbnail | 1280x720 | Judge at ~120px wide — that's search-result size |
| Presentation/blog header | 1920x1080 | 16:9 default |

Design at 2x when the target is raster; export per platform rather than letting platforms recompress one master.

## Thumbnails Specifically

- 3 elements maximum: face or subject, 2-4 word text, one background. Every published guide beyond that is noise at 120px.
- Text on thumbnails is display type: 700+ weight, extreme contrast (white with dark outline or block background), never body-style text.
- The subject's gaze or motion vector points at the text or into the frame (`layout.md` composition).

## Banners and Ads

- Fixed-size constraint flips the workflow: set the canvas, place the CTA and logo in their standard corners (logo top-left or bottom-right, CTA bottom-right in LTR), then fit the message in what remains.
- Animated banners: 3 frames max (hook → value → CTA), end on the CTA frame, total loop under ~15s per common ad-platform limits.
- Design the smallest required size first (e.g. 300x250); scaling up is easy, cramming down destroys the hierarchy.

## Export Hygiene

- Photos → JPEG/WebP; flat graphics and text → PNG/SVG. JPEG text gets halo artifacts; PNG photos get huge.
- sRGB color profile on export — unmanaged wide-gamut exports shift colors on most screens.
- Text as vector or high-res raster (2x); 1x rasterized text is the blurry-thumbnail tell.
- Check the artifact on a phone screen in a real feed context before delivering, not only full-screen on desktop.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Filling the canvas because the space exists | One-glance artifacts need dominance, not coverage | One message at a third of the area, whitespace does the rest |
| All info at equal size ("everything matters") | Nothing is readable at feed size | One huge line, one small cluster |
| Text over a busy photo region | Fails contrast at the worst pixel | Scrim, blur the region, or relocate text to a clean zone |
| Reusing a landscape design for stories by scaling | Message lands in the cropped/overlaid zones | Recompose per aspect ratio |
| Neon accent on neon background | Equal lightness = invisible at a glance | Force a lightness gap first, hue second |
