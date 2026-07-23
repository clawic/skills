# Critique — Symptom to Cause When It "Looks Off"

Users report feelings, not causes. Each chain below is ordered by probability; every step is a check against a rule, not a taste call. Deliver findings as a ranked punch-list (worst first, each item = symptom → violated rule → concrete fix) unless `critique style` in config says otherwise.

## The Universal First Three

1. Grayscale the artifact. If the 1-3 ranking is not obvious in 3 seconds, the problem is hierarchy — fix that before touching color or type.
2. Squint (or blur at 10% zoom). Exactly the title and one action should survive. Zero survivors = flat hierarchy; three+ survivors = competing rank-1s.
3. Count vertical alignment lines. More than 3 = the "messy" feeling has a measurable cause (Core Rule 6).

## "Cluttered / busy / overwhelming"

1. Count borders, boxes, and background fills — each is noise. Delete containers; restore grouping with gaps (between-group >= 2x within-group).
2. Spacing audit: gaps off the base scale, or equal gaps everywhere, destroy grouping and read as chaos even with few elements.
3. Count typefaces, weights, and colors. More than 2 families, 3 weights, or 1 accent hue: consolidate before removing content.
4. Only after 1-3: cut content. Ranking inventory reveals rank-3 items that can collapse into a "details" layer.

## "Amateur / unprofessional / cheap"

Ordered by how often each is the actual cause in generated or non-designer work:

1. Misaligned edges — elements 2-5px off a shared line. Snap everything to <= 3 alignment lines.
2. Mixed radii, mixed shadows, or mixed icon styles on sibling elements — one scale per artifact.
3. Default-stack fonts mixed with a styled font, or faux bold/italic (browser-synthesized) — load the real weights (`fonts.md`).
4. Pure #000 on #FFF plus fully saturated accent colors — near-black text, tinted neutrals, one desaturated accent.
5. Stretched or off-ratio images and logos — crop, never distort.
6. Centered long text with ragged widths — left-align (Core Rule 6).

## "Flat / boring / nothing stands out"

1. Everything is body-sized: no element uses a full step in any channel. Promote the one rank-1 by size x1.25 twice (e.g. 16 → 25) or weight 400 → 700.
2. All same lightness: demote rank-3 to 40-60% gray on white; contrast between levels creates energy for free.
3. Whitespace is uniform: compress within groups, expand between them — rhythm reads as intentional design.
4. Still flat: the content itself has no ranking (ten equal bullets). Fix the content: pull one number or claim out as the hero.

## "Unbalanced / lopsided / feels wrong"

1. Visual weight audit: large image one side, small text block the other. Balance by mass, not by count — one photo outweighs three paragraphs.
2. Focal point sits dead-center vertically: optical center is slightly above geometric center; content placed mathematically centered reads as sinking.
3. Margins unequal without intent: outer margins must match or follow a deliberate asymmetry (e.g. wide left caption column).
4. One orphaned element floats off-grid: attach it to a group or delete it.

## "Hard to read"

1. Measure: over 75ch (wide container) or under 45ch (over-narrow column) — fix the container (560-600px for prose) before the font.
2. Line-height under 1.5 at body size, or over ~1.8 (lines drift apart).
3. Contrast under 4.5:1 — light gray text on white is the top offender; 40-60% gray is for rank-3 demotion only, never long body text.
4. Justified text with rivers, or tracked lowercase body — reset to left-aligned, default tracking.
5. Text over an image without a scrim: add a gradient overlay until contrast passes 4.5:1 at the worst point.

## "Colors feel off / dirty / clashing"

1. Pure gray neutrals beside a saturated accent — tint neutrals 2-5% with the accent hue.
2. Two accent hues competing at equal saturation — demote one to a tint or drop it (60-30-10).
3. Accent used on non-interactive decoration — the accent must mean something (action/state), or it is noise.
4. Semantic collision: brand red next to error red, or green decoration near success states — shift the non-semantic use (`palettes.md`).
5. Hues at mismatched lightness (neon yellow next to navy at full strength) — rebuild both from the same lightness ramp.

## "Too corporate / generic / AI-looking"

1. Symptom of defaults: purple-blue gradient, Inter, glassmorphism cards, emoji bullets — replace with one deliberate choice per channel (a real typeface with character, one specific hue, flat surfaces).
2. Every section identical width and rhythm — vary section backgrounds or break the grid once per page, deliberately.
3. Stock-feeling imagery — prefer real product shots, data, or typography-led sections over decorative illustrations.

## "It's fine but not premium"

Premium = restraint executed precisely: fewer elements, larger whitespace (1.5-2x your current section gaps), one typeface used at strong size contrast, muted palette with one confident accent, zero decoration that isn't information. Audit what to remove, not what to add.

## When You Are Truly Stuck

Rebuild the view from the content inventory alone: plain text, ranked 1-3, on the type scale with default spacing — no color, no boxes. If the plain version reads better than the styled one, the styling was the problem; re-add one decision at a time until it degrades. The step that breaks it names the file to open next.
