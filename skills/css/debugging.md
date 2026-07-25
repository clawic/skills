# Debugging — Symptom to Cause

Work symptom-first. Each chain is ordered by probability, and every step is a check you can run in DevTools in under a minute, not a guess. Never add a property until one of these steps names the mechanism.

## The Universal First Three

1. **Is the rule matching at all?** Styles panel: a rule that isn't listed never matched (bad selector, wrong element, sheet not loaded); a rule listed with strikethrough matched and lost. Two different bugs, opposite fixes.
2. **Read Computed, not Styles.** Computed shows the value that actually applies after inheritance, clamping (`min-width`/`max-width`), and layout negotiation. The authored `width: 300px` computing to `250px` is a constraint, not a cascade problem.
3. **See real boxes.** Turn on the flex/grid overlays and the layout badges; or `* { outline: 1px solid red }` temporarily — outline never affects layout, which is why it beats border here.

## Style Not Applying

1. Rule absent from Styles → selector never matched. Check in order: typo in the class, the element you inspected is not the one styled (children of a component often carry the class), the sheet failed to load (Network tab), or the file is excluded by the build.
2. Rule present but struck through → it lost. Hover the winner: same origin? Then the order is layers → specificity → source order. "It worked before the refactor" is almost always a layer or import-order change.
3. Rule present, not struck, no visual change → the value is invalid (DevTools shows a warning triangle) or unsupported, or it's a no-op on this element: `width` on an inline element, `gap` on a block parent, `z-index` without positioning, `align-items` on a non-flex/grid box.
4. Value there in Computed, still wrong on screen → something later in the pipeline overrides the pixels: a transform, a parent's `overflow: clip`, a `filter`, or a pseudo-element painted over it.
5. Custom property resolving to nothing → the variable is undeclared at this element (custom properties inherit, they are not global) or invalid at computed-value time — a declared-but-wrong-typed value falls back to inherit/initial, never to the `var()` fallback.

## Element Invisible or Not Where Expected

| Check | Command or tell |
|---|---|
| Zero size | Computed shows 0×0 — empty block, `height: 100%` with an unsized parent, or a grid/flex child that collapsed |
| Painted behind something | Stacking context trap (SKILL.md — Stacking Contexts) |
| Clipped away | Ancestor with `overflow: hidden`/`clip`, or a `clip-path` on a parent |
| Off-screen | Negative margins, an absolute element with no positioned ancestor (it anchored to the viewport/initial containing block) |
| Transparent | `opacity: 0`, `color: transparent`, or a `filter` from a parent |
| Present in DOM, absent in layout | `display: none` (removed) vs `visibility: hidden` (space kept) vs `content-visibility: hidden` (skipped, size kept) |
| `position: fixed` behaving like absolute | An ancestor has `transform`, `filter`, `perspective`, `backdrop-filter`, or `will-change` on any of those |

## Unexpected Horizontal Scroll

There is always one concrete culprit; find it before styling.

1. In the console: `[...document.querySelectorAll('*')].filter(el => el.getBoundingClientRect().right > document.documentElement.clientWidth)` — the last elements in that list are the ones sticking out.
2. Usual causes, in order: a fixed-width child (`width: 600px` inside a narrower parent), `100vw` (includes the scrollbar width on desktop — use `100%`), negative margins on a full-bleed section, a long unbroken string (URL, hash, code token), an absolutely positioned decoration with no clipping ancestor, a grid with a `1fr` track that refuses to shrink (`minmax(0, 1fr)`).
3. Do not "fix" it with `overflow-x: hidden` on `body` or `html`: that creates a scroll container on the root, which breaks `position: sticky` descendants and hides the real bug from the next person.
4. When step 1 names a culprit you cannot remove (third-party embed, a decorative element that must overhang), the containment fix is `html { overflow-x: clip }` (Safari 16+): `clip` creates no scroll container, so sticky and anchor scrolling keep working, and `overflow-x: clip` with `overflow-y: visible` is a legal pair that stays as authored. Reach for it only after the culprit is identified — it still hides the overflow from the next reader.

## Spacing Is Wrong

- Gap appears where you set none → margin collapse (block layout only), or the parent's `gap` plus a child's margin adding up.
- Space refuses to shrink → padding on the parent, `min-height` on the child, or line-height leading around a single-line text node.
- Space at the top of the page nobody wrote → first child's `margin-top` escaping through a parent with no border/padding (`display: flow-root` on the parent).
- Uneven grid gutters → `gap` is between tracks only; outer spacing is the parent's padding.
- Inline-block elements with a mystery ~4px gap → the whitespace text node between tags; use flex/grid, not `font-size: 0` hacks.

## Works on Desktop, Broken on Mobile

| Symptom | Cause |
|---|---|
| Section bottom hidden behind browser UI | `100vh` — use `100svh` |
| Page zooms when tapping an input | Input `font-size` below 16px on iOS |
| Hover styles stick after tapping | Touch fires hover; wrap hover styling in `@media (hover: hover)` |
| Fixed footer jumps when the keyboard opens | Visual viewport shrinks; anchor to the layout viewport and test with the keyboard open |
| Content under the notch or home bar | Missing `viewport-fit=cover` + `env(safe-area-inset-*)` padding |
| Tiny tap targets that "work in the emulator" | Below the 24×24 / 44×44 floor (SKILL.md — Accessibility Floor) |
| Horizontal scroll only on phones | A fixed-width element narrower viewports can't absorb (chain above) |

## Works in Chrome, Not in Safari or Firefox

1. Is the property supported at all? Check the feature, not the browser: an unsupported declaration is dropped without warning, and the Styles panel shows it struck or greyed.
2. Is it a whole-rule kill? One unrecognized selector in a comma-separated list invalidates the ENTIRE list in every engine — `::-webkit-slider-thumb, ::-moz-range-thumb {}` styles nothing anywhere. Split into one rule per engine.
3. Prefix or different spelling needed? `backdrop-filter` and the `mask` shorthand still want `-webkit-` in older Safari, and form controls need `appearance: none` AND `-webkit-appearance: none` there.
4. Same CSS, different result → engine-level behavior difference (flex percentage heights, sticky in tables, form control internals), not a bug in your file.

## Layout Jumps While Loading

1. Record a Performance trace with the Layout Shift lane, or open the Web Vitals overlay: it names the shifting element, which usually ends the investigation.
2. Ranked causes: media without reserved space (`aspect-ratio` or width/height attributes), web font swap with unmatched metrics, late-injected banners/ads above content, a scrollbar appearing on load (`scrollbar-gutter: stable`), and client-rendered content replacing a differently-sized skeleton.
3. Font-driven shift is fixed with `font-display: swap` plus metric overrides (`size-adjust`, `ascent-override`, `descent-override`) on the fallback `@font-face`, not with `font-display: optional`.

## Animation or Transition Does Nothing

1. Transitioning from an unset or discrete value → there is no start value to interpolate (`display`, `height: auto`, an unregistered custom property); discrete properties need `transition-behavior: allow-discrete`.
2. The element was just inserted → the browser never saw the start state; needs `@starting-style`.
3. Property is not animatable, or you animated a shorthand that resolves differently in the two states.
4. `prefers-reduced-motion` is on in the OS and your media query is doing exactly what you asked.
5. It plays but stutters → layout-triggering property in the frame loop (SKILL.md Core Rule 2) or too many composited layers (`performance.md`).

## Styles Break Only in Production

- Class name present in the DOM but absent from the CSS → the purge/tree-shaking step dropped it because it was built dynamically (`bg-${color}`). Safelist it or write full class names.
- Order changed after bundling → sheets concatenated in a different order than the dev server served them; declare `@layer` order in a file that always loads first.
- Rule exists in the file, missing from the shipped bundle → critical-CSS extraction or a minifier dropping "unused" at-rules; diff the built file, not the source.
- Only broken on first paint → the critical inline CSS and the full sheet disagree; the full sheet arriving later is the flash.
- Only broken for some users → a browser-extension stylesheet or forced-colors/high-contrast mode.

## When You Are Truly Stuck

Delete, don't add. Copy the broken component into an empty page with no stylesheet, confirm it works, then re-add your CSS one declaration block at a time — the block that breaks it names the mechanism and the file to open next. Binary-search a large sheet the same way by disabling half its layers.
