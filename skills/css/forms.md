# Form Controls

Styling native inputs without losing their behavior, and showing validation state at the right moment. Focus-ring and target-size rules are in SKILL.md — Accessibility Floor.

## How Far You Can Restyle Each Control

| Control | Fully styleable | Reality |
|---|---|---|
| text, email, password, textarea | Yes | Plain boxes; only autofill and the reveal buttons are UA-controlled |
| checkbox, radio | Color only, or full rebuild | `accent-color` recolors in one line; full control needs `appearance: none` + your own box and checkmark |
| range | Track and thumb | Needs per-engine pseudo-elements in SEPARATE rules |
| select (closed box) | Yes | Arrow via `appearance: none` + background image |
| select (open list) | No, portably | Option lists are OS widgets; `appearance: base-select` is Chromium-experimental. A fully styled listbox means a custom component with real keyboard support |
| file | Button only | `::file-selector-button` is standard; the filename text is not styleable |
| date, time, color | Field only | Internal pickers are UA widgets, webkit pseudo-elements only |
| progress, meter | Mostly | `accent-color` first; per-engine pseudo-elements after |

Default: recolor with `accent-color` and style the box. Rebuild a control only when the design genuinely requires it — every rebuild inherits keyboard, IME, autofill, and screen-reader work the browser was doing for free.

## The Vendor-Pseudo Rule That Breaks Everything

One unrecognized selector invalidates the ENTIRE comma-separated list, in every engine:

```css
/* styles nothing, anywhere */
input[type=range]::-webkit-slider-thumb,
input[type=range]::-moz-range-thumb { background: var(--accent); }

/* correct: one rule per engine */
input[type=range]::-webkit-slider-thumb { background: var(--accent); }
input[type=range]::-moz-range-thumb     { background: var(--accent); }
```

Same trap with `::-ms-*` leftovers and `:-moz-*` pseudo-classes. Any rule that does nothing in all browsers, with no console warning, is this bug.

## Validation State at the Right Time

- `:invalid` matches an untouched empty required field on first paint — a form that is red before the user types. Use `:user-invalid` (matches only after interaction or a submit attempt) and `:user-valid` for the success state. All engines since late 2023.
- `:placeholder-shown` powers float labels and "empty" styling without JS: `.field:has(input:placeholder-shown) label { … }`.
- Error text must be tied to the field with `aria-describedby` — CSS state is invisible to screen readers. Color alone never communicates the error: pair it with an icon or text.
- Style the error container with `:has()` so the whole field group reacts: `.field:has(:user-invalid) { --border: var(--danger) }`.
- `:required`/`:optional` for the indicator; do not use the asterisk as the only signal.

## Autofill

- Browsers force their own background on autofilled fields and ignore `background-color`. The portable workaround is an inset shadow plus text-fill color:

```css
input:-webkit-autofill {
  box-shadow: inset 0 0 0 100px var(--surface);
  -webkit-text-fill-color: var(--text);
}
input:autofill { border-color: var(--accent); } /* standard selector */
```

- The autofill transition hack (`transition: background-color 100000s`) is a delay, not a fix — it flashes on slow paints.
- Do not disable autofill for convenience; it is an accessibility and security feature. Style around it.

## Layout and Sizing

- `input`, `select`, and `textarea` do not inherit font by default: without `font: inherit; color: inherit` on the controls, every field renders in the UA's system face at ~13px while the page around it uses your stack. It is a base-layer declaration, not a per-component one.
- `box-sizing: border-box` globally, or a 100%-width input with padding overflows its grid cell.
- iOS zooms the page when focusing an input whose font-size is below 16px. Set 16px minimum on mobile-visible inputs, or accept the zoom.
- `field-sizing: content` grows a textarea or input with its content and removes the classic mirror-div JS (Chromium first — treat it as an enhancement over a `rows` default).
- Textareas need `resize: vertical` — free horizontal resize breaks every layout it lives in.
- `fieldset` accepts `display: grid` in current engines; if a legacy target is in scope, wrap the contents in a div and lay that out.
- Number inputs: hide spinners with `appearance: textfield` plus `::-webkit-outer-spin-button { appearance: none }` — and prefer `inputmode="numeric"` on text inputs for codes and quantities.

## Buttons

- Reset the UA styles explicitly (`background: none; border: 0; font: inherit; cursor: pointer`) rather than fighting them per-instance.
- Keep the hit area at the accessibility floor even when the visual is small: pad, or expand with a `::after { position: absolute; inset: -8px }` overlay.
- Disabled buttons take no pointer events, so no tooltip fires and no focus lands — for anything the user might ask "why?" about, keep it enabled and explain on submit, or add `aria-disabled` with your own guard.
- `:active` and `:focus-visible` both need styles; a button with only `:hover` is unusable by keyboard.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `:invalid` for error styling | Fires before interaction | `:user-invalid` |
| Combining `-webkit-` and `-moz-` pseudos in one selector list | Whole rule invalid in both | Separate rules |
| `outline: none` on inputs | Removes the only keyboard affordance | Restyle `:focus-visible` |
| Custom select rebuilt with divs | Loses keyboard, typeahead, mobile picker, form submission | Style the native box; rebuild only with full ARIA and testing |
| Placeholder as label | Disappears on typing, fails contrast, unreadable for screen readers when used as the accessible name | Real `<label>`, placeholder for examples only |
| 14px inputs on mobile | iOS zooms on focus and the layout jumps | 16px minimum |
| Styling `::placeholder` to near-invisible grey | Fails 4.5:1 and hides the hint | Keep placeholders above the contrast floor |
