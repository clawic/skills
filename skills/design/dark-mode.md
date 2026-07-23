# Dark Mode — Theming, Not Inversion

Dark mode is a second palette derived from the same ramp (`palettes.md`), not a filter. Every rule in SKILL.md still applies; the gates run once per theme.

## Surfaces and Elevation

- Base surface #121212-class (10-12% lightness), not pure #000 by default — see SKILL.md Where Experts Disagree for the OLED exception.
- Shadows barely read on dark: elevation is communicated by LIGHTER surfaces instead. Each raised layer steps up in lightness (e.g. base 12% → card 16% → dialog 20%); Material formalizes this as overlay increments.
- Keep 3 surface levels maximum; more lightness steps than that and the levels blur together.
- Borders replace shadows for subtle separation: 1px at ~20-25% white opacity.

## Text and Contrast

- Text is white at reduced opacity, not pure #FFF: ~87% primary, ~60% secondary, ~38% disabled (Material's scale). Pure white on dark halates — strokes bloom and thin fonts smear.
- Contrast floors are unchanged (4.5:1 body, 3:1 large/UI) but the failure direction flips: light-mode mid-grays that passed on white often fail on #121212. Re-check every text/surface pair; never assume the light values transfer.
- Slightly increase line-height or use a half-step heavier weight if the typeface renders thinner on dark (common with light text on dark due to halation).

## Color Adaptation

- Desaturate accents 10-20% and lighten them one ramp step: saturated light-mode accents vibrate against dark surfaces and often fail contrast.
- Semantic colors need dark variants too: light-mode error red (~45% lightness) is illegible on #121212 — use the 60-70% lightness step of the same hue.
- Large tinted fields flip direction: light mode uses 95% lightness tints for section backgrounds; dark mode uses 15-18% lightness tints of the same hue.
- Brand colors that cannot shift (logo rules) sit on a slightly lighter surface chip rather than being altered.

## Imagery and Media

- Illustrations and diagrams designed on white need a dark variant or a subtle light backing plate; transparent PNGs with dark strokes vanish.
- Photos: reduce brightness slightly (5-10%) so they don't glow against dark chrome; pure-white product shots get a soft dark treatment or a card.
- Charts: re-derive gridlines (white at low opacity) and re-check series colors on the dark surface (`data-viz.md`).

## Implementation Hygiene

- Both themes come from one token set with two values per token; hardcoded hex anywhere guarantees a missed spot (the white flash, the unreadable tooltip). Token architecture at scale: `design-system` skill.
- Honor the OS preference (`prefers-color-scheme`) as the default; a manual toggle stores an override.
- Test the seams: focus rings, selection color, scrollbars, form autofill backgrounds, and loading skeletons — the five spots that stay light-mode by accident.
- Email dark mode is a different, hostile problem (forced inversion): `email.md`.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| CSS invert filter or naive color swap | Images invert, shadows invert, semantics break | Derive a dark palette from the ramp |
| Pure #000 surface + pure #FFF text | Maximum halation; harsh, tiring | 12% surface, 87% white text |
| Same accent hex in both themes | Vibrates and/or fails contrast on dark | Desaturate 10-20%, lighten one step |
| Elevation via darker shadows | Invisible on dark surfaces | Lighter surface per raised level |
| Checking contrast only in light mode | Mid-grays fail silently on dark | Run Output Gates once per theme |
