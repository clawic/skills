# Fonts — Choosing, Pairing, Setting Type

Deep type craft (optical sizes, variable axes, print settings) lives in the `typography` skill; this file covers picking and executing type for one artifact. If `brand_file` is set, its typefaces override the selection tables here.

## Selection by Context

| Artifact | Default | Why |
|---|---|---|
| Product UI | System stack or one neutral grotesque (e.g. Inter-class) | Zero load cost, native feel, tabular figures available |
| Marketing / landing | One display face with character + neutral text sans | Type carries the brand voice; body stays invisible |
| Editorial / long-form | Serif text face at 18-20px, 1.6 line-height | Serifs aid long-read flow; bump size with the measure |
| Slides | One sans, weights 400/700 only | Projectors blur fine serifs; bold survives distance |
| Code / data | Monospace with slashed zero | 0/O and 1/l ambiguity is a correctness bug |
| Print documents | Text serif for body, sans for headings | Print resolution rewards serifs; roles stay distinct |

System stack: `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif` — the zero-cost default for product work.

## Pairing Rules

- Pair by role contrast: display serif + text sans, or geometric display + humanist text. Two similar sans faces read as a rendering mistake — if they're close, use one.
- Maximum 2 families per artifact. A superfamily (matching serif + sans from one designer) is the safe pairing when unsure.
- The text face decides readability; choose it first at real body size with a real paragraph, then find a display face that contrasts with it.
- Weights: load 400 and 700 (or 400/600 for UI). Skip adjacent weights — 400 vs 500 is indistinguishable at body sizes. Never let the renderer synthesize bold or italic (faux weights distort letterforms); load the real cuts.

## Setting Type

- Body: 16px floor, 1.5 line-height, 45-75ch measure (66ch target, Bringhurst). These three interlock — widen the measure, add line-height.
- Headings: tighten line-height to 1.1-1.3 and letterspacing slightly negative (-1% to -2%) above ~31px; large type looks loose at default tracking.
- ALL CAPS only for 1-2 word labels, +5-10% letterspacing. Never track lowercase body text.
- Numbers: tabular (fixed-width) figures in tables, timers, and prices in lists — proportional figures make columns jitter. `font-variant-numeric: tabular-nums`.
- Hierarchy through the scale (`type_scale_ratio`), not ad-hoc sizes: every size on the page comes from the ladder (16/20/25/31/39 at 1.25). An off-ladder size is a bug, not a nuance.
- Emphasis inside body text: italic for standard emphasis, bold sparingly for scannable keywords, never underline (reads as link) and never color alone.
- CJK text: no italics (fake obliques distort ideographs), line-height 1.7-2.0, and measure counts characters not ch units.

## Loading (web)

- WOFF2 only; subset to the scripts actually used. One variable font replaces four static weight files and unlocks in-between weights.
- `font-display: swap` so text renders immediately in the fallback; pick a fallback with similar metrics to cut layout shift on swap.
- Self-host over third-party CDN links: removes a request chain and a privacy leak.
- Load at most: 1-2 families x 2 weights + italic where genuinely used. Every extra cut delays first render.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Display font used for body text | Display cuts have tight spacing tuned for large sizes; illegible at 16px | Text cut for body, display for 25px+ |
| Trendy font for a data-heavy UI | Ambiguous numerals and no tabular figures corrupt scanning | Neutral grotesque with tabular figures |
| Justified body text on the web | Browsers don't hyphenate well; rivers of whitespace | Left-aligned, ragged right |
| Mixing font sources (one Google, one system, one image) | Metrics and rendering differ; page looks assembled | One source, defined stack, real cuts loaded |
| Thin weights (100-300) for UI text | Sub-pixel rendering breaks strokes on low-DPI screens | 400 minimum for text sizes; thin only at display sizes |
