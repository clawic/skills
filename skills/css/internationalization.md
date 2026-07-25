# RTL, CJK, and Logical Properties

Writing the stylesheet so a language switch is a `dir` attribute, not a fork. Canonical home for logical properties, RTL mirroring, text expansion, and per-script fonts.

## Logical Properties Map

| Physical | Logical | Notes |
|---|---|---|
| `margin-left` / `right` | `margin-inline-start` / `-end` | Flips with direction |
| `padding-top` / `bottom` | `padding-block-start` / `-end` | Flips in vertical writing modes |
| `left` / `right` | `inset-inline-start` / `-end` | Also `inset-inline: 0` shorthand |
| `width` / `height` | `inline-size` / `block-size` | Plus `min-`/`max-` variants |
| `text-align: left` | `text-align: start` | The single most common RTL bug |
| `border-left` | `border-inline-start` | Shorthands exist for both axes |
| `border-radius` corners | `border-start-start-radius` etc. | Order is block-then-inline |
| `float: left` | `float: inline-start` | Same for `clear` |

- Adopt them from day one when RTL or vertical writing is remotely possible: retrofitting is a whole-file rewrite, and half-converted files are worse than either pure form.
- Never mix physical and logical on the same box side; the two systems resolve independently and last-write-wins is unreadable in review.
- Flex and grid are already logical: `flex-direction: row`, `justify-content: start`, and grid column order follow direction automatically. `row-reverse` does NOT mean "RTL" — it reverses within the current direction.

## What `dir="rtl"` Does Not Flip

- **Transforms.** `translateX(20px)` moves right in both directions. Flip the sign under `[dir="rtl"]`, or express the offset with `inset-inline-start` instead.
- **Directional icons.** Back/forward arrows, undo, reply, and progress chevrons must mirror: `[dir="rtl"] .icon-back { scale: -1 1 }`. Icons with no direction (search, settings, download) must NOT flip, and neither must logos or media playback controls (play always points to the timeline's forward direction, which is still right in most players — decide per product and be consistent).
- **Numbers, code, URLs, phone numbers.** They stay LTR inside RTL text. Wrap with `dir="ltr"` plus `unicode-bidi: isolate`, or use `<bdi>` for user-generated names — otherwise a trailing punctuation mark jumps to the wrong side.
- **Shadows and gradients.** `box-shadow: 4px 4px` and `linear-gradient(to right, …)` are physical; mirror them explicitly when the design is directional.
- **Scroll direction.** `scrollLeft` is negative or reversed in RTL depending on engine — a JS concern, but it surfaces as "the carousel starts at the wrong end".

## Text That Grows

- Translations expand: German and Finnish commonly run about 30% longer than English, and short UI labels can double or more. Localization guidance generally recommends budgeting the most expansion for the shortest strings.
- Consequences for CSS: no fixed widths on buttons, tabs, or nav items; no `white-space: nowrap` without a truncation plan; no two-line assumptions in headings; icon-plus-label layouts must survive wrapping.
- Test with a pseudo-locale (real translations padded, or accented duplicates) before the strings arrive — it catches the same bugs weeks earlier.
- Vertical space also grows: Thai, Hindi, and Vietnamese need more line-height for stacked marks; a tight `line-height: 1.2` clips diacritics.

## CJK and Line Breaking

- CJK breaks between characters, so `overflow-wrap` is rarely needed, but `line-break: strict` prevents breaking before closing punctuation and small kana.
- Korean breaks at spaces: `word-break: keep-all` prevents mid-word breaks that are legal by default and look wrong.
- `text-spacing-trim: trim-start` (Chromium first) removes the visual gap from full-width opening punctuation at line starts.
- Set generous line-height (1.7-1.8 is common for CJK body text) and avoid `letter-spacing` on CJK: it separates characters that read as a unit.
- Emphasis is `text-emphasis: dot`, not italics — CJK faces usually have no true italic, and synthesized slanting looks broken (`font-synthesis: none` makes that visible in review).

## Fonts Per Script

- List script-specific families before the generic fallback; the browser picks per character, so `font-family: Inter, "Noto Sans Arabic", sans-serif` uses Inter for Latin and the Arabic face for Arabic glyphs.
- `unicode-range` in `@font-face` splits a family so each script's file downloads only when that script appears — the difference between a 40KB subset and a full CJK font of several megabytes.
- Arabic and Indic scripts need larger optical sizes to stay legible: raising `font-size` by roughly 10% for those scripts via `:lang()` is a standard adjustment.
- `lang` on the html element (and on mixed-language spans) drives font selection, hyphenation dictionaries, quote glyphs (`q`), and `:lang()` styling. Missing `lang` disables hyphenation everywhere, with nothing in the console.

## Testing RTL

1. Set `dir="rtl"` on `<html>` and read the page as a whole: what stayed put is what was written physically.
2. Check the layout edges: sidebars, drawers, dropdown alignment, badge positions, close buttons.
3. Check mixed content: a Latin brand name inside Arabic text, a phone number, a code snippet.
4. Check the icons list above.
5. `:dir(rtl)` is the selector for direction-conditional rules; `[dir="rtl"]` works everywhere and is safer when direction is set explicitly.
