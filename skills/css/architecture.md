# Stylesheet Architecture

Organizing CSS so the cascade works for you: layers, scoping, nesting, component boundaries, and migrating off preprocessors. `authoring_mode` decides which half of this file applies; `browser_support` gates `@scope` and bare-element nesting.

## Layer Strategy

Declare the order once, in the first file the bundler emits:

```css
@layer reset, tokens, base, layout, components, utilities;
```

- Later `@layer` blocks append to the named layer; the order fixed by that first line never changes, so import order stops mattering — the single biggest source of "works in dev, broken in prod" style bugs.
- Third-party CSS goes in an early layer: `@import url("vendor.css") layer(vendor);` — now a single-class rule of yours beats their `#id .deep .selector` without `!important`.
- Unlayered styles beat every layer. Two defensible uses: your final overrides, and nothing else. Treat an unlayered ruleset in a component file as a bug.
- Utilities last so a `.mt-0` wins over a component rule of any specificity — this is what makes utility classes safe next to handwritten CSS.
- `!important` inverts layer order (earliest layer wins), so an `!important` in `reset` is nearly unbeatable. Use once, deliberately, for invariants like `[hidden] { display: none !important }`.
- `authoring_mode` picks the ordering job here: `plain-css`/`sass` own the whole list above; `tailwind` means the framework owns `utilities` and your handwritten rules go in `components` or earlier, never unlayered; `css-in-js` means the extracted sheet still needs one `@layer` declaration emitted before any component styles, or injection order decides the cascade at runtime.

## Scoping Without Naming Conventions

- `@scope (.card) { img { … } }` limits a rule to a subtree, so plain element selectors become safe inside a component.
- Donut scoping — `@scope (.card) to (.card__content)` — excludes slotted or user content, which is the one thing BEM never solved.
- Scope proximity is a cascade step of its own: between two matching rules of equal specificity, the one whose scope root is nearer the element wins. That is how a nested theme beats an outer one without specificity games.
- `:scope` refers to the scope root; the implicit `&` inside `@scope` behaves like `:scope`.
- Support is broad but not universal; when `browser_support: legacy`, keep a class-based convention and treat `@scope` as an enhancement.

## Nesting Without Specificity Creep

- This whole section is about NATIVE nesting (`authoring_mode: plain-css`, `css-in-js`, or a `tailwind` sheet with a nesting plugin). Under `authoring_mode: sass`, `&` is text substitution and none of the specificity mechanics below fire — which is exactly why they surprise teams mid-migration.
- `&` compiles to `:is(<parent selector list>)`, and `:is()` takes the specificity of its most specific argument. Nesting inside `.btn, #cta` gives every nested rule ID-level specificity — a nesting-only bug that never happened in Sass, whose `&` was pure text substitution.
- Depth limit 3, and never nest a whole component's internals: deep nesting recreates the descendant-selector coupling that classes were meant to remove.
- Nest for states and variants (`&:hover`, `&[aria-expanded="true"]`, `&.is-loading`) and for media/container queries; not for DOM structure.
- Native nesting requires `&` for compound selectors and now accepts bare element selectors in current engines; older parsers do not — check `browser_support` before shipping bare element nesting.

## Component Boundaries

- A component styles its inside only: no `margin`, `width`, `position`, or `grid-area` on the root (SKILL.md Core Rule 8). The parent layout supplies placement, usually through `gap`.
- Expose a deliberate API with custom properties instead of letting consumers reach in with descendant selectors: `.card { --card-pad: 1rem; padding: var(--card-pad) }`. Register them with `@property` when the type matters.
- Variants as data attributes (`[data-variant="ghost"]`) keep specificity flat and are visible in the DOM inspector; modifier classes work too — pick one per codebase.
- One file per component, named after the component. A `utils.css` that grows past a screen is where dead CSS goes to hide.
- No `!important` inside components. If a third-party style forces one, the fix belongs in a dedicated override layer, with a comment naming the offender.

## Shadow DOM and Web Components

- Page styles do not reach inside a shadow root, and shadow styles do not leak out. What DOES cross: inherited properties (`color`, `font`, `line-height`) and every custom property.
- That makes custom properties the theming API for web components: consume `var(--accent, …)` inside, and consumers theme without piercing.
- `::part(name)` exposes specific internals for styling; `:host`, `:host(.variant)`, and `:host-context()` style the element itself (`:host-context` is not implemented everywhere — avoid depending on it).
- `::slotted(selector)` matches only TOP-LEVEL slotted nodes, never their descendants — the most common surprise when styling slotted content.
- Prefer `adoptedStyleSheets` (constructable stylesheets) over a `<style>` per instance: one sheet shared by every instance instead of N copies parsed separately.

## Migrating Off Sass

| Sass feature | Native replacement | Caveat |
|---|---|---|
| `$variables` | Custom properties | Runtime and inheritable — a feature, but they cannot be used in media query conditions |
| Nesting | Native nesting | `&` carries `:is()` specificity (above) |
| `@mixin` / `@include` | Utility class, or a custom-property "recipe" | No native mixins; repeated declaration blocks are the honest answer |
| `@extend` | Nothing — and that is fine | It always produced surprising selector explosions |
| `lighten()` / `darken()` | `color-mix()` and relative color syntax | Perceptual results differ; re-tune the palette |
| `@use` / partials | `@layer` + bundler imports | Never ship raw `@import` — each level costs a round trip |
| Build-time math | `calc()`, `min()`, `max()`, `clamp()` | Runtime math cannot be used where the parser needs a literal (media query values) |

Migrate leaf-first: tokens, then components, then layout. A half-migrated file where Sass variables and custom properties both define the same color is worse than either. Flip `authoring_mode` from `sass` to `plain-css` only when the last `.scss` partial is gone — while both exist, emitted examples must match the file being edited, not the destination.

## Keeping It Clean

- Dead CSS: DevTools Coverage identifies unused rules on the pages you visit — a lower bound, not proof. Deleting needs the route matrix, not one page load.
- Purge tools cannot see `class={`bg-${x}`}`: dynamic class names must be safelisted or written in full, or they vanish only in production.
- Review checks worth automating: no `!important` outside the override layer, no IDs in selectors, no hardcoded hex in components, nesting depth ≤3, no `@import` in shipped CSS.
- The specificity ceiling those reviews enforce comes from `naming_convention`: `bem` and `css-modules` cap at one class (`.card__title`, hashed class); `utility` caps at zero component-level overrides — a fix belongs in the markup or the utility layer; `none` (the default) still caps at single-class, since nothing else guarantees a consumer can override you. Anything above the ceiling needs a comment naming why.
- Size budget: a stylesheet growing faster than the component count means duplication, usually variants pasted rather than parameterized.
