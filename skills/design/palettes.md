# Palettes — Color Without a Design System

Full color systems, tokens, and cross-surface validation live in the `color` skill; this file derives a working palette for one artifact. If `brand_file` is set, start from its colors instead of deriving.

## Derivation Procedure

1. Pick ONE accent hue from the content's meaning (finance → restrained blue/green range; food → warm range) or the user's brand. Everything else derives from it.
2. Build a 9-step lightness ramp of that hue (roughly 95/90/80/70/60/50/40/30/20% lightness). Keep hue constant; in OKLCH/HSLuv the perceived lightness stays honest across hues — plain HSL lies (yellow at 50% HSL lightness is far lighter than blue at 50%).
3. Tint the neutrals: grays carry 2-5% of the accent's saturation. Pure gray next to a saturated accent looks dirty (SKILL.md Traps).
4. Assign by 60-30-10: ~60% neutral surfaces, ~30% secondary (tinted neutrals, muted mid-steps), ~10% accent — and the accent marks interactive or state-bearing elements only.
5. Derive states from the ramp, not from new hues: hover = one step darker, active = two steps, disabled = 40% opacity of the base or the 80% lightness step with muted text.
6. Contrast-check every produced pair against the floors (4.5:1 body, 3:1 large/UI; 7:1 and 4.5:1 when `accessibility_target` is AAA). A ramp step that fails against its intended background gets adjusted in lightness, not swapped in hue.

## Semantic Colors

- Reserved meanings: red = destructive/error, green = success, amber/yellow = warning, blue = information. Do not spend them on decoration.
- Each semantic color needs its own mini-ramp (background tint at ~95% lightness, border mid-step, text step passing 4.5:1 on the tint) — a lone #FF0000 has nowhere to go for backgrounds and borders.
- Brand collides with semantics (red brand, green brand): shift the semantic shade AND add an icon; hue alone never carries state (Core Rule 7).

## Multi-Color Needs

- A second accent is justified only by a second persistent dimension (e.g. "income vs expenses" throughout). Choose it at the same lightness and saturation as the first, ~120-180 degrees away in hue; then both stay subordinate to 60-30-10.
- Categorical sets (tags, calendars, avatars): sample 5-7 hues at ONE fixed lightness/saturation so no category looks more important; chart series cap at 6 and follow their own rules (`data-viz.md`).
- Gradients: two neighbors on the hue wheel (blue→violet), never complements (blue→orange muddies at the midpoint). One gradient per view, on a display element, never behind body text without a contrast check at the worst point.

## Backgrounds and Surfaces

- Page background off-white (98-99% lightness, tinted) beats #FFF for large surfaces; cards can then sit at #FFF with no border needed.
- Text on photos: add a scrim (bottom gradient, 0 → 60% black) until the worst point passes 4.5:1; measure at the lightest pixel under the text, not the average.
- Large saturated fields tire the eye: full-bleed accent sections use the 90-95% lightness step with accent-colored text, not the 50% step as wall paint.

## Dark Variant

Map the ramp, don't invert it: surfaces use the 10-20% lightness steps (never pure #000 by default), text drops to ~87% white, accents desaturate 10-20% and lighten one step to keep contrast. Full treatment: `dark-mode.md`.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Picking 5 hues first, assigning later | No hierarchy; everything competes | One hue → ramp → 60-30-10 assignment |
| HSL lightness treated as perceived lightness | Yellows blow out, blues go black at "equal" L | Derive in OKLCH, or check every pair's contrast ratio |
| Accent for headings, links, icons, AND decorations | Accent stops meaning "interactive" | Accent = action/state only; headings are near-black |
| New hue invented per state (hover teal on blue) | Palette sprawls, states look unrelated | States = ramp steps of the base hue |
| Copying a palette site's 5 swatches verbatim | Swatches lack surface/text/state steps; contrast unverified | Use the swatch as the seed hue, run the procedure |
