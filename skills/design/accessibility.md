# Accessibility — Contrast, CVD, Motion, Focus

Design-time execution rules. A full WCAG audit with screen-reader passes belongs to the `accessibility-audit` skill; this file keeps the artifact from needing one twice.

## Contrast Floors

- AA (default `accessibility_target`): 4.5:1 body text, 3:1 large text (24px+, or 19px bold) and UI component boundaries (input borders, icons, focus rings against adjacent colors).
- AAA: 7:1 body, 4.5:1 large — required in some government/health contexts; expect it to eliminate most mid-gray text.
- Measure at the worst point: text over images/gradients passes at the lightest pixel under the text, not the average (`palettes.md` scrim rule).
- Placeholder text, disabled labels users must read, and "subtle" captions are the chronic offenders — 40-60% gray demotion (Hierarchy Procedure) is for rank-3 metadata, and even it must clear 4.5:1 if the user needs to read it.

## Color Vision Deficiency

- ~8% of men, ~0.5% of women have CVD; red/green confusion (deutan/protan) dominates. Red-vs-green as the only difference — status dots, diff views, chart series — is the single most common failure.
- Rule: hue never carries meaning alone (Core Rule 7). Pair color with an icon, label, weight, or position; vary lightness between any two meaning-bearing hues so they differ in grayscale.
- Test: grayscale the artifact (existing gate) plus a deuteranopia simulation for charts and status systems specifically.

## Motion and Animation

- Honor `prefers-reduced-motion`: replace movement with opacity fades; never remove the state change itself, only the motion.
- Parallax, auto-playing video, and large moving backgrounds can trigger vestibular symptoms — reserve them for user-initiated contexts and give a pause control.
- Nothing flashes more than 3 times per second (seizure threshold, WCAG 2.3.1).
- Auto-advancing content (carousels, tickers) needs pause/stop controls — and usually should not auto-advance at all (`landing-pages.md`).

## Interaction Targets and Focus

- Pointer targets: 24x24 CSS px is the WCAG 2.2 hard floor; platform guidance stays 44pt (iOS) / 48dp (Android) for touch — design to the platform number, treat 24px as the never-below line.
- Focus visible on every interactive element: a 2px ring offset from the element, 3:1 contrast against the adjacent background. Removing outlines without replacement makes keyboard use impossible.
- Focus order follows visual order; a layout that LOOKS right but tabs diagonally is a design bug, not just a code bug — check after any absolute-positioning trick.
- Links inside body text: underline or another non-color cue in addition to color; color-only links fail CVD and low-vision scanning.

## Text and Structure

- Body text must survive 200% zoom without loss: fixed-height text containers and px-locked layouts clip; min-height and relative units don't.
- Line-height 1.5 at body size (already Core Rule 4) is also the accessibility floor; tight line-height hits low-vision and dyslexic readers first.
- Real text over text-in-images, everywhere (also `email.md`): images of text can't reflow, resize, or be read aloud.
- Heading levels are structure, not styling: one h1, no skipped levels; visual size comes from the scale, semantic level from the outline.
- Alt text: describe function for functional images ("Search"), content for informational ones, empty alt for pure decoration — a screen reader reading "decorative-blob-3.png" is design debt.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Light gray "elegant" body text (#999 on white) | 2.8:1 — fails everyone, not just low vision | Demote by size/weight/position; keep >= 4.5:1 |
| Red/green-only status system | Invisible to deutan/protan users | Icon + color; lightness gap between the hues |
| `outline: none` for aesthetics | Keyboard users lose their position entirely | Styled focus ring, 2px, 3:1 against surroundings |
| Meaning in hover-only tooltips | No touch or keyboard equivalent | Visible label or tap/focus-triggered disclosure |
| Animation as the only state feedback | Invisible under reduced-motion | State also encoded in color/text/icon; motion is garnish |
