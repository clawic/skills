# Typography Mechanics

Text sizing, spacing, and breaking: line-height, measure, units, variable fonts, wrapping and truncation.

## Line Height and Vertical Metrics

- Always unitless. `line-height: 1.5` inherits the RATIO; `150%` and `24px` inherit the computed value, so a 40px heading inherits the body's 24px leading and its lines overlap. This is the single most common typography bug in inherited stylesheets.
- Practical ratios: 1.5-1.6 for body text (WCAG 1.4.12 requires content to survive 1.5 spacing), 1.1-1.25 for display headings, 1.4 for UI labels. Longer measures need more leading; short buttons need less.
- Half-leading is why a button's text never looks centered: the extra leading splits above and below the glyphs, and cap height sits off-center inside it. `text-box: trim-both cap alphabetic` removes it (Chromium and Safari first; keep padding-based centering as the fallback).
- `line-height` on a flex/grid container does not center its text — the box centers, the glyphs still carry leading. Center with `place-items: center` plus trimmed leading, not with padding-top nudges.

## Measure and Sizing Units

- Body measure 45-75 characters (`max-width: 65ch` is the working default). `ch` is the width of `0` in the current font, so the same value gives different results per font — check, don't assume.
- Unit map: `rem` for type and spacing (respects user font-size); `em` for things that must scale with their own text (icon size, badge padding, letter-spacing); `px` for hairlines and shadows; `ch` for measure; `cap`/`lh` where supported for optical alignment.
- Never set `html { font-size: 62.5% }` to make rem math easy — it overrides the user's browser font-size preference for the whole document. If the team wants 10px math, use `rem_base` in config and convert honestly.
- Minimum readable body size on screen is 16px; 14px is acceptable for dense UI chrome, below 12px only for non-essential metadata (and never for anything a user must read to act).

## Font Stacks and Variable Fonts

- `font-family: system-ui, sans-serif` gets the OS UI face with zero download; add explicit fallbacks per script when non-Latin content is in scope.
- Variable fonts pay off from about three weights or when you need optical sizing; below that, two static WOFF2 subsets are usually smaller.
- Drive axes through high-level properties (`font-weight: 550`, `font-stretch: 87.5%`) rather than `font-variation-settings` — the low-level property does not inherit or animate predictably and disables the browser's synthesis rules.
- `font-optical-sizing: auto` is on by default for fonts with an `opsz` axis; turn it off only to match a design that was drawn at one size.
- Faux bold and italic (`font-synthesis`) appear when the family lacks the face — set `font-synthesis: none` in brand-critical contexts so the mistake becomes visible instead of ugly.

## Numerals and Small Details

- `font-variant-numeric: tabular-nums` on tables, timers, prices, and anything that updates in place — proportional digits make numbers jitter as they change.
- `font-variant-numeric: oldstyle-nums` for running prose in serif faces; lining figures for UI.
- Letter-spacing: negative tracking on large headings (about -0.01em to -0.03em) because type designed for text is too loose at display sizes; positive tracking (+0.05em) on uppercase and small caps; never track body text below 16px.
- `text-transform: uppercase` changes rendering only — the DOM text and clipboard content stay lowercase, which matters for names and codes.
- Avoid `text-rendering: optimizeLegibility`: it forces kerning and ligature work on large blocks and has shipped rendering regressions; it is not a quality switch.

## Wrapping, Breaking, Truncation

| Need | Declaration | Note |
|---|---|---|
| Even heading lines | `text-wrap: balance` | Engines cap the work (Chromium at 6 lines) — safe on all headings, ignored on long text |
| No single-word last line in paragraphs | `text-wrap: pretty` | Chromium first; degrades to normal wrapping |
| Long URLs and hashes must not overflow | `overflow-wrap: anywhere` | `word-break: break-all` breaks even short words — wrong for prose |
| Hyphenated justification | `hyphens: auto` + `lang` attribute | Without `lang` the dictionary never loads and nothing hyphenates |
| Keep a name together | `white-space: nowrap` on the span | Combine with `overflow-wrap` on the parent |
| Preserve user line breaks | `white-space: pre-wrap` | Only place `pre-wrap` belongs is user-generated text |
| One-line ellipsis | `overflow: hidden; white-space: nowrap; text-overflow: ellipsis` + a width source | In flex, add `min-width: 0` |
| Multi-line clamp | `display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 3; overflow: hidden` | Prefixed but works in every current engine |

- Ellipsis truncation removes information with no affordance: pair it with a `title` or a tooltip when the full string matters (file names, emails).
- `hyphenate-limit-chars: 6 3 3` stops the two-letter fragments that make auto-hyphenation look cheap.

## Rich Text You Do Not Control

CMS and user content needs its own guardrails, applied once on the container:

- Constrain media: `img, video, table { max-width: 100% }` and `overflow-wrap: anywhere` on the container.
- Wrap tables in a scroll container — pasted tables are the top source of horizontal scroll on article pages.
- Spacing via `> * + *` flow spacing (`margin-block-start: 1em`) rather than per-element margins, so unexpected element orders still space correctly.
- Headings arriving at the wrong level are a structure problem, not a CSS one — style by class where you can, and never renumber with CSS `content`.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `line-height: 150%` | Inherits the computed pixel value into headings | Unitless ratio |
| `html { font-size: 62.5% }` | Silently shrinks text for users who raised their default size | Keep 100%, convert with `rem_base` |
| `word-break: break-word` | Deprecated alias with confusing behavior | `overflow-wrap: anywhere` |
| Fixed heights on text containers | Translation and user font-size overflow the box | `min-height` plus padding |
| Icon sized in `px` beside `em` text | Icon stays put while text scales with zoom | Size icons in `em`, align with `vertical-align` or flex |
| Centering text with `padding-top` | Breaks the moment the line count or font changes | Flex/grid centering plus leading trim |
