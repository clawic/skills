# Cross-Engine Compatibility

Where the three engines actually differ, how to gate features, and the two hostile environments (forced colors, HTML email). Feature availability checked 2026-07; re-verify anything marked as trailing before relying on it.

## Feature Gating

- `@supports (property: value) { … }` tests the parser, not the behavior. Write the fallback OUTSIDE the query and the enhancement inside, so an engine that ignores both still renders.
- `@supports selector(:has(a))` for selectors, `@supports not (…)` for the inverse branch. Never test a browser: an unknown engine tomorrow gets the right branch from a feature test and the wrong one from UA sniffing.
- Declaration-level fallback is free: two declarations of the same property, the older one first. The engine keeps the last one it understands.
- Custom properties break that pattern — `color: var(--x)` is always parseable, so a bad value at computed-value time falls back to inherit/initial, not to the previous declaration. Type them with `@property` when a fallback must be reliable.
- `browser_support` config drives the policy: `evergreen` ships current features bare; `widely-available` waits for the feature to be interoperable across all three engines for a while; `legacy` adds prefixed fallbacks and blocks Chromium-first features.

## Chromium-First Features (as of 2026-07)

Ship these as enhancements with a working non-enhanced state, and re-check before promoting them to defaults:

- `interpolate-size: allow-keywords` / `calc-size()` — animating to `auto`; the `grid-template-rows: 0fr → 1fr` trick stays the portable path.
- Scroll-driven animations (`animation-timeline: scroll()`/`view()`) — build so the no-timeline state is the final state, then it degrades to "already visible".
- `field-sizing: content` — auto-growing inputs; the fallback is a `rows` default.
- `@container style()` queries beyond custom-property equality.
- `appearance: base-select` and full select-list styling; the portable ceiling is styling the closed box only.
- Anchor positioning is further along (Chromium first, Safari shipped, Firefox trailing) — still gate it with `@supports (position-area: block-end)` and keep an absolute-positioned fallback.
- View transitions: same-document is broadly available, cross-document is not — treat cross-document as progressive enhancement.

## Recurring Safari and WebKit Differences

- iOS viewport: `100vh` includes the retractable browser UI — `svh`/`dvh` exist for this.
- iOS zooms on focus when an input's font-size is below 16px.
- Rounded `overflow: hidden` plus a transform can jitter or leak corners; `clip-path: inset(0 round …)` is steadier.
- `backdrop-filter` still benefits from the `-webkit-` prefix in older Safari; same for `mask` shorthand parts.
- Form controls need `appearance: none` AND `-webkit-appearance: none` when legacy Safari is in scope.
- Momentum scrolling is default now; `-webkit-overflow-scrolling: touch` is dead code — delete it.
- Percentage heights inside flex columns resolve differently from Chromium in edge cases; give the flex parent a definite size instead of chaining percentages.
- Date and time inputs render their own internals with webkit pseudo-elements; there is no portable way to restyle the picker.

## Firefox Notes

- Scrollbar styling has always been the standard properties (`scrollbar-width`, `scrollbar-color`); `::-webkit-scrollbar` does nothing there.
- `:host-context()` is not implemented — do not build theming on it; custom properties cross the shadow boundary and are the portable API.
- Range and progress internals use `::-moz-*` pseudo-elements, which must live in their own rules.
- Text-wrapping refinements (`text-wrap: pretty`) landed later than in Chromium; the fallback is ordinary wrapping, which is safe.

## Forced Colors and High Contrast

- `@media (forced-colors: active)` replaces your palette with the user's system colors: backgrounds, text, links, and buttons all become system keywords. Anything whose only affordance was a background color disappears.
- Check in that mode: icon-only buttons, focus rings, custom checkboxes, chart legends, and dividers built from background gradients.
- `forced-color-adjust: none` per element for genuine color swatches; never on whole components.
- Related but distinct: `prefers-contrast: more` is a request for stronger separation while your palette stays in control.

## HTML Email

The extreme compatibility case. Rules here are the opposite of everything else in this skill:

- Outlook on Windows renders with the Word engine: no flexbox, no grid, no `position`, unreliable `max-width`, no `background-size`. Layout means nested `<table>` elements with fixed widths.
- Inline styles are the baseline; `<style>` blocks in `<head>` survive in many clients but are stripped by some (notably parts of Gmail's mobile paths), so treat them as enhancement only. Media queries live there and simply do not run where the block is dropped.
- Web fonts are unavailable in several major clients: choose a system stack that degrades gracefully and set fallbacks explicitly.
- Dark mode is partly client-controlled: Apple Mail honors `prefers-color-scheme`, several clients force their own inversion regardless. Design so a forced inversion is survivable — avoid images with baked-in white backgrounds, use PNGs with real transparency, keep logos legible on both.
- Fixed pixel widths around 600px remain the safe container; single-column layouts break least.
- Test in the actual clients (an email-rendering service or real accounts). No emulator predicts Outlook.

## Deciding Whether to Care

- Feature availability answers "can I use it"; your analytics answer "must I". A feature missing from an engine with a fraction of a percent of your traffic and a graceful fallback is not a blocker.
- Distinguish cosmetic degradation (no blur behind the panel) from functional failure (menu positioned off-screen). Only the second needs a polyfill or a library.
- Write down the support target with the user's `browser_support` preference so the next decision is not re-litigated per feature.
