# Text — Resizing, Type Mechanics, and Real Content

Text is where a design meets reality: the string is longer than the mock, the font is not licensed, the line height does not match the browser. Almost every text bug in Figma is a resizing mode or a unit mismatch.

## Resizing Modes

| Mode | Behavior | Right for | Fails when |
|---|---|---|---|
| Auto width | Box grows horizontally, never wraps | Buttons, labels, chips, single-word tags | A long string pushes the layout sideways forever |
| Auto height | Width fixed by the parent, height grows with wraps | Body copy, descriptions, anything multi-line | Nothing — this is the default for real content |
| Fixed size | Both dimensions typed | A precisely reserved slot | The string is longer than the mock — silent clipping |

Default: Auto height inside a Fill container. Auto width only when the content is genuinely one short token. Fixed size is almost always an accident, and it is the single most common cause of "the text is cut off in production".

## Truncation

- Max lines with an ellipsis is the deliberate way to limit text; it tells the engineer the intent (`line-clamp`).
- Fixed height that happens to cut text is not truncation — it clips mid-glyph with no ellipsis and no signal.
- Truncating in the design without specifying the tooltip or expand behavior transfers the decision to the engineer. Annotate what happens to the hidden content.
- Test truncation with a string that ends mid-word, not with one that happens to end cleanly.

## Line Height, Spacing, and the Browser Mismatch

- Line height accepts px, percentage, or Auto. Auto uses the font's own metrics and differs per family, which is why a design and its build drift by a few pixels.
- Browsers default to a unitless line height. Percentage in Figma maps to unitless in CSS (150% → `line-height: 1.5`); px maps to px. Pick one convention per system and put it in the token names.
- **Vertical trim** removes the half-leading above and below the text box, so the box's edges sit on the cap height and baseline. With it on, the padding you set in Figma equals the padding in the build. Without it, every button in the system is a few pixels taller than specced.
- Letter spacing in percentage scales with size (correct for a type ramp); in px it does not (correct for a single fixed treatment).
- Paragraph spacing is a property of the text node, not a gap between two text layers. Two stacked text nodes with an auto layout gap is the sturdier structure for handoff.

## Text Styles vs Variables

- Type ramps still live in text styles: a style bundles family, size, weight, line height, letter spacing and case in one applied object.
- Variables cover the numeric parts and the string content; full binding of every typographic property has lagged color. Check the current coverage before promising a type ramp built purely from variables.
- Practical setup: a text style per ramp step (`Heading / L`, `Body / M`, `Caption`), and number variables feeding the size and line-height values inside those styles where binding is available.
- Name ramp steps by role, not by size. `Heading / L` survives a scale change; `Text 24/32` does not.

## Fonts

- Fonts installed locally need the desktop app or the font helper; the browser alone sees only Google Fonts and the org's shared fonts.
- A missing font shows a banner and substitutes silently in the render — a file opened by a teammate without the license looks subtly wrong and nobody notices until handoff. Fix by restricting the system to licensed families and stating them on the cover page.
- Variable fonts expose weight, width, optical size and slant as continuous axes. Use named instances for the type ramp; free-floating axis values are unreproducible in code unless the axis is exposed in CSS.
- Web-font loading is the engineer's problem but the designer's constraint: every extra family and weight is a network request and a FOUT risk. A system with more than a handful of family-weight combinations is a performance decision nobody made deliberately.

## Real Content

- `Lorem ipsum` hides the real length distribution. Use real strings, and always include the longest one the field permits.
- The three strings every text field needs tested: the shortest realistic, the longest permitted, and one with an unusual shape (a hyphenated surname, an all-caps acronym, a language with long compounds).
- Numbers in tables and dashboards need tabular figures (the `tnum` OpenType feature) or the columns visibly jitter between values. Proportional figures are correct in prose and wrong in any column of numbers.
- Localization inflates: expansion of 30% or more is routine for short UI strings going from English into German or Finnish, and it lands hardest on buttons and tabs — the elements most often set to Fixed size.
- RTL support has been partial and has moved; verify the current state before committing to an Arabic or Hebrew build inside Figma rather than in code.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Text cut off with no ellipsis | Fixed size text node | Auto height, Hug parent |
| Ellipsis appears where it should not | Max lines set on the node | Remove or raise Max lines |
| Layout stretches sideways forever | Auto width on a long string | Auto height plus a max-width clamp |
| Build is a few px taller than the design | Vertical trim off | Turn it on and re-derive padding |
| Line height differs between design and code | Auto line height, or px vs unitless mismatch | Fix the convention; state it in the token names |
| Type looks wrong on a teammate's machine | Missing font substituted silently | Restrict to licensed families; list them on the cover |
| Numbers jitter between rows | Proportional figures | Enable tabular figures on the numeric style |
| German build breaks every button | Tested only on English strings | Re-test with a 30%+ expanded string |
| Weight not reproducible in code | Free-floating variable-font axis value | Use a named instance the CSS can request |
