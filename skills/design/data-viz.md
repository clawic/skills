# Data Viz — Charts That Answer a Question

A chart exists to answer one question; the question picks the chart, the encoding, and what gets deleted. Chart placement in dashboards: `ui-screens.md`.

## Chart Choice by Question

| Question | Chart | Rule |
|---|---|---|
| How does it change over time? | Line | Time on x; no zero-baseline requirement, but declare a truncated axis |
| How do categories compare? | Horizontal bar | Sorted by value (unless the axis is ordinal); zero baseline mandatory |
| What are the parts of a whole? | Stacked bar or pie | Pie only for <= 5 slices summing to 100%; otherwise bar |
| How do two measures relate? | Scatter | Add a trend line only when the claim IS the trend |
| How is it distributed? | Histogram / box plot | Bin count changes the story — try 2-3 widths before publishing |
| Where does it concentrate? | Heatmap | Sequential ramp, ordered axes |
| Flow between states? | Sankey | Only when flows are the finding; else a table |

Default when unsure: horizontal bar — labels stay readable and comparison is by position, the encoding humans judge most accurately (Cleveland's ranking: position > length > angle > area > color).

## Integrity Rules

- Bar charts start at zero — bar length IS the value; a truncated bar axis fabricates differences. Line charts may zoom, with the axis range visible.
- One y-axis. Dual axes let arbitrary scaling manufacture any correlation; use two stacked panels sharing the x-axis instead.
- Sort categorical bars by value; alphabetical order hides the ranking the reader came for.
- Aspect ratio shapes perception of slope: pick it so the trend reads honestly, not dramatically; wider flattens, taller exaggerates.
- Label the units and the time window on the chart itself; a chart that migrates (slide, screenshot, chat) must carry its own context.

## Declutter (order of deletion)

1. Chart junk: 3D, shadows, gradients on bars, background fills — delete all, always.
2. Borders and heavy gridlines → light gray horizontal gridlines only (or none for direct-labeled bars).
3. Legend → direct labels at line ends or on bars whenever series <= 6; a legend forces eye round-trips.
4. Axis tick density → 4-6 ticks; abbreviated numbers (12k, 1.4M) over raw digits.
5. Redundant axis titles when the chart title already names the measure.

The chart title states the finding, not the topic: "Signups doubled after the pricing change", not "Signups over time" — same assertion rule as `slides.md`.

## Color in Charts

- Magnitude → one-hue sequential lightness ramp; signed/divergent data → diverging ramp through a neutral midpoint at the meaningful zero (`palettes.md` ramp mechanics).
- Categorical series: distinguishable hues at equal lightness, 6 maximum — beyond that, group into "other" or split into small multiples.
- Emphasis pattern: the series under discussion in the accent color, all others gray. This single move turns a spaghetti line chart into an argument.
- Never encode by hue alone: vary lightness so the chart survives grayscale and CVD (Core Rule 7); check red/green pairs specifically (`accessibility.md`).
- Semantic consistency across a document: the same entity keeps the same color in every chart.

## Text in Charts

- All chart text horizontal; rotated x-labels mean the wrong chart orientation — flip to horizontal bars.
- Chart body text >= 12px on screen (20pt+ on slides); axis labels are content, not decoration.
- Annotate the finding on the chart (arrow + short phrase at the anomaly) — the annotation layer is where analysis lives.

## Small Multiples

Comparing across many categories or periods: repeat one simple chart per category on shared axes rather than overlaying 12 series. Shared scale is mandatory — per-panel scales silently break comparison.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Pie with 8 slices | Angle comparison fails beyond a few parts | Sorted horizontal bar |
| Rainbow categorical palette | Implies ordering that doesn't exist; CVD hostile | Equal-lightness hues <= 6, or gray + accent |
| Smoothed/interpolated lines for sparse data | Invents data between real points | Straight segments + visible point markers |
| Truncated bar axis to "show the difference" | Fabricates the difference visually | Zero baseline; zoom with a line chart or show deltas |
| Legend with 10 entries | Reader plays memory matching | Direct labels, small multiples, or gray + accent |
| Stock spreadsheet default theme | Gridlines, borders, tilted labels — all noise | Run the declutter deletion order |
