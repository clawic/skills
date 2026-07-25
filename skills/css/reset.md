# Reset and Base Layer

What belongs in `@layer reset` and `@layer base` before any component CSS, what only pretends to be a reset, and the order to write a new stylesheet in. Every line below prevents a bug you would otherwise rediscover.

## The Reset, Whole

```css
@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
  * { margin: 0; }

  html {
    -webkit-text-size-adjust: 100%;
    text-size-adjust: 100%;
    scrollbar-gutter: stable;
    tab-size: 4;
  }

  body { min-height: 100svh; line-height: 1.5; }

  :where(img, picture, video, canvas, svg) { display: block; max-width: 100%; height: auto; }
  :where(input, button, textarea, select) { font: inherit; color: inherit; }
  :where(p, h1, h2, h3, h4, h5, h6) { overflow-wrap: break-word; }
  :where(ul, ol)[role="list"] { list-style: none; padding-inline-start: 0; }

  :focus-visible { outline: 2px solid; outline-offset: 2px; }
  [hidden] { display: none !important; }
}
```

## Why Each Line Is There

- `box-sizing` needs `*::before, *::after` spelled out — pseudo-elements are not covered by `*`. The alternative (`html { box-sizing: border-box }` + `*, *::before, *::after { box-sizing: inherit }`) exists so one embedded third-party subtree can opt out with a single declaration; pay that indirection only if you actually host foreign markup.
- `* { margin: 0 }` deletes the whole margin-collapse bug class (the "why did my container move down" one) and makes spacing an explicit `gap` or flow rule. It also zeroes paragraph and heading margins — the base layer puts back the ones the design wants.
- `text-size-adjust: 100%` stops mobile engines inflating text through their own boost heuristics ("my phone type is bigger than I set"). The `-webkit-` twin is still required.
- `scrollbar-gutter: stable` on `html` — the document scroller is `<html>`, not `<body>`. Reserving the gutter is the cheapest layout-shift fix in the file: a page that grows past one screen stops jumping sideways.
- `min-height: 100svh` on body: `svh` is the viewport unit that never hides behind mobile browser UI, so a full-height shell is correct on the first render instead of after a resize.
- `line-height: 1.5` unitless on body: the ratio inherits, so headings can override with their own tighter ratio; a percentage or px value here poisons every descendant with the body's computed leading.
- `img { display: block }` removes the inline baseline gap — the mystery ~4px strip under every image. `max-width: 100%` stops an oversized image from setting the page's minimum width; `height: auto` keeps the `width`/`height` attributes from squashing it once the width is constrained, while the attributes still give the browser the aspect ratio that reserves space before load.
- Form controls do not inherit `font` or `color`. Without this line every input renders in the UA's ~13px system face next to your type stack.
- `overflow-wrap: break-word` on text blocks: a pasted URL or hash can no longer widen the page. Deliberately not `anywhere` — `anywhere` also shrinks min-content size, which changes flex and grid track sizing; `break-word` only breaks at paint time.
- The list reset is role-qualified on purpose: `list-style: none` makes Safari/VoiceOver stop announcing the element as a list. Requiring `role="list"` in the markup keeps the semantics and makes the removal intentional. Unqualified `ul { list-style: none }` is the accessibility bug hiding inside most resets.
- `:focus-visible { outline: 2px solid }` with no color keeps `currentColor`, so the ring stays legible on any themed surface; `outline-offset` lifts it clear of filled controls. A reset that ships `outline: none` is a reset that ships an accessibility failure.
- `[hidden] { display: none !important }` is the one deliberate `!important`: any `display:` on a component otherwise beats the attribute, and the attribute is how frameworks and scripts hide things. Placed in the earliest layer, where `!important` inverts layer order, it is the invariant nothing can accidentally outrank.
- `:where()` around the opinionated rows makes them zero-specificity: a consumer's single class always wins and nobody reaches for `!important` to escape the reset.

## Reset vs Base

- **Reset** neutralizes UA differences and footguns. Zero design decisions, zero brand, no numbers a designer would argue about.
- **Base** is your document default: type scale on `body`, heading rhythm, link styling, `color-scheme: light dark` on `:root`, `text-wrap: balance` on headings, default table borders and `border-collapse`.
- The test: if deleting the declaration changes the design, it belongs in base. Mixing them means a rebrand has to edit the reset, which is how resets rot into 400-line frameworks.
- Both live in declared layers. Unlayered author styles beat every layer, so a reset written unlayered wins fights it should always lose.

## What Does Not Belong

- A full normalize.css: most of it patches engines nobody ships in 2026, and the parts still worth keeping are in the block above.
- `html { font-size: 62.5% }` — it overrides the user's browser font-size preference to save you a division.
- `* { padding: 0 }` — it flattens list indentation and `fieldset`/`legend` structure everywhere; scope padding resets to the elements you actually restyle.
- `* { animation: none !important }` as a reduced-motion policy: killing animations outright means `animationend`/`transitionend` never fire and components that clean up in those handlers hang. Motion is opt-in in the base layer (SKILL.md — Accessibility Floor).
- `-webkit-font-smoothing: antialiased` is a taste call, not a reset: it thins glyphs on macOS, weakens light-on-dark text, and does nothing on Windows. Ship it only if a designer asked and both themes were checked.
- `button { cursor: pointer }` is contested — native buttons use the default arrow, and platform-faithful teams keep it that way. Decide once in base; never per component.
- Tag-level typography (`h1 { font-size: 2rem }`) — that is design, so it is base, not reset.

## Starting a New Stylesheet

1. Layer order declaration, first line the bundler emits: `@layer reset, tokens, base, layout, components, utilities;`.
2. The reset block above, in `@layer reset`.
3. Tokens: primitives then semantics, plus `color-scheme: light dark` on `:root` so UA widgets follow the theme.
4. Base: the element defaults that carry the design.
5. Components; utilities last so a utility can always win.
6. Nothing unlayered except your final, deliberate overrides.
