# Selectors and Specificity

Cascade mechanics that decide fights, and the selectors that replace JS.

## The Cascade, Actual Order

Precedence: origin and importance → cascade layers → specificity → scope proximity → source order. Specificity is step THREE — most "specificity problems" are really layer problems. Consequences:

- Unlayered author styles beat ALL layered styles. Put resets, frameworks, and third-party CSS in layers; keep your overrides unlayered or in a final layer — overrides then win at any specificity.
- Declare order once, first line of the sheet: `@layer reset, base, components, utilities;` — later declarations of the same layer append, order stays fixed.
- `!important` REVERSES layer order: among important declarations, the EARLIEST layer wins. An `!important` inside your reset layer is nearly unbeatable — that is a feature for guaranteed invariants and a footgun everywhere else.
- Specificity is the triple (ids, classes, types) compared position by position — the folklore "100/10/1" model is wrong at the edge: eleven classes never beat one ID.
- Scope proximity only breaks ties between equal-specificity rules: the rule whose scope root is the nearer ancestor wins, which is how a nested theme overrides an outer one without extra classes.
- Inline `style` attributes sit above every author rule and below author `!important`. Full top of the order: running transitions beat everything, then UA `!important`, then user `!important`, then author `!important`, then running animations, then normal author declarations — which is why an `!important` freezes an animated property mid-flight but cannot stop a transition.

## Specificity Tools

- `:where(...)` — always zero specificity. Wrap library/reset selectors in it so consumers override with a single class.
- `:is(...)` and `:not(...)` — specificity of the MOST specific argument, even for the non-matching ones: `:is(#nav, .item)` is ID-level everywhere it applies.
- Keep component selectors at single-class specificity; reserve IDs for fragment targets and JS, never styling.

## :has() Patterns

- `form:has(:invalid)` — form-level error state without JS mirroring.
- `label:has(input:checked)` — styled selectable cards/chips with zero scripts.
- `.item:has(+ .active)` — the element BEFORE the active one (previous-sibling selection).
- `.grid:has(.card:hover) .card:not(:hover)` — dim siblings of the hovered card.
- Cost model: `:has()` triggers ancestor re-checking on DOM mutation. Cheap on components; expensive as `body:has(...)` on large, frequently-mutating documents. Scope the subject as narrowly as possible.
- `:has()` cannot nest inside `:has()`.

## nth-child

- `:nth-child(An+B of S)` counts only elements matching S: `:nth-child(odd of .visible)` keeps zebra stripes correct with hidden rows — the classic filter-breaks-stripes bug. All engines since 2023.
- `:nth-child` counts every sibling; `:nth-of-type` counts same-tag siblings. With mixed content (headings between cards), `of S` is usually what you actually meant.

## Custom Properties, the Cascade Gotchas

- "Invalid at computed-value time": `color: var(--c, red)` with `--c: 12px` declared → the fallback does NOT apply; the property computes to inherit/initial instead. Fallback fires only when `--c` is UNDECLARED. Guarantee types with `@property --c { syntax: "<color>"; inherits: true; initial-value: red; }`.
- Custom properties are invisible to `@media` and `@supports` conditions; they DO drive `@container style(...)` queries.
- Unregistered properties animate discretely (snap at 50%); registering with `@property` and a syntax enables real interpolation — this is the animated-gradient enabler.
- `--x: ;` (empty value) is valid and truthy for fallback purposes — basis of the space-toggle hack; document it if you use it, it reads as a typo.

## Focus Selectors

- `:focus-visible` = keyboard/programmatic focus only — style this, keep mouse clicks clean. Never bare `outline: none` (floor rules: SKILL.md — Accessibility Floor).
- `:focus-within` on the container — elevate a whole form group while any field inside has focus.

## Pseudo-Elements

- `::before`/`::after` need a `content` value to exist at all (`content: ""` counts) and are children of the element, inside its box — they cannot escape its clipping or its stacking context.
- They do not apply to replaced elements: no pseudo-elements on `img`, `input`, `iframe`, `br`, or `video`. The workaround is a wrapper.
- Decorative content only: screen readers may or may not announce generated text, and it never survives copy-paste or translation. Anything meaningful goes in the DOM. `content: "→" / ""` supplies an empty alt string so it is skipped cleanly.
- `content` also accepts `attr()` (strings only) and images: `content: url(check.svg) / "done"`.
- The invisible-click-target pattern: `position: relative` on the parent, `::after { position: absolute; inset: 0 }` to stretch a link over a card — nothing else inside stays clickable, which is the tradeoff.
- Others worth styling: `::marker` (list bullets and counters — only `color`, `font`, and `content` apply), `::selection` (set both color and background or you get one against an unknown other), `::first-line` and `::first-letter` (limited property sets; `::first-letter` is the drop-cap tool), `::placeholder` and `::file-selector-button` on form controls, `::backdrop` behind top-layer elements.

## Counters

- `counter-reset: step` on the container, `counter-increment: step` on each item, `content: counter(step) ". "` in a `::before` — numbered steps that renumber automatically when items are added, filtered, or reordered.
- `counters(step, ".")` produces nested numbering (1.2.3) from a self-nesting structure.
- Counters count elements you increment, not visible ones: skip hidden items explicitly (`[hidden] { counter-increment: none }`) or the sequence gains gaps.
- Never number semantic lists with counters when the number carries meaning — an `<ol>` already exposes position to assistive tech.

## Negation and Matching Gotchas

- `:not(.a, .b)` takes the specificity of the most specific argument, so `:not(#x)` is ID-level. Wrap in `:where()` to neutralize: `:where(:not(#x))`.
- `:not()` matches elements that lack the class INCLUDING html and body when unqualified — always anchor the subject (`.item:not(.active)`, not `:not(.active)`).
- `* + *` (the "lobotomized owl") spaces siblings regardless of type; it also hits elements you forgot exist (script tags render nothing but still count as siblings in some frameworks' output).
- `:empty` matches only elements with no children AND no text, whitespace included — comments are ignored but a newline is not.
- Case sensitivity: class and id selectors are case-sensitive in standards mode; attribute VALUES are case-sensitive unless you add the `i` flag; type selectors in HTML are not.

## Attribute Selectors Worth Knowing

- `[href^="http"]:not([href*="yourdomain"])` — external-link marking in pure CSS.
- `[data-state="open" i]` — case-insensitive flag `i` when the DOM value casing is out of your control.
- `[class*="col-"]` — matching generated/utility class families; cheap despite folklore, but prefer a shared real class when you own the markup.
