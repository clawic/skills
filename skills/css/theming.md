# Theming and Design Tokens

Token architecture, dark mode, and multi-brand. The contrast floor a theme must clear is in SKILL.md — Accessibility Floor.

## Three Token Layers, One Rule

1. **Primitive** — the raw scale: `--blue-500`, `--space-4`, `--radius-md`. No meaning, never referenced by a component.
2. **Semantic** — the contract: `--surface`, `--surface-raised`, `--text`, `--text-muted`, `--border`, `--accent`, `--danger`. Defined in terms of primitives, and the ONLY layer components read.
3. **Component** — local overrides: `--button-bg: var(--accent)`. Optional; earns its place when a component must be themeable from outside.

The rule that makes it work: a component that references a primitive has broken the theme. Grep for `--blue-` in component files as a review check — every hit is a future dark-mode bug.

## Dark Mode Without Duplicated Blocks

```css
:root {
  color-scheme: light dark;
  --surface: light-dark(#ffffff, #16181c);
  --text:    light-dark(#16181c, #e9eaec);
}
:root[data-theme="light"] { color-scheme: light; }
:root[data-theme="dark"]  { color-scheme: dark;  }
```

- `light-dark()` reads `color-scheme`, so one declaration per token covers both modes, and the manual toggle works by flipping `color-scheme` alone — no second block of variables to keep in sync. All engines since 2024.
- `color-scheme` also switches the UA's own surfaces: form controls, scrollbars, and `::selection`. Omitting it is why a "dark" page still has white dropdowns.
- Legacy path when `browser_support: legacy`: define tokens in `:root`, override inside `@media (prefers-color-scheme: dark)` AND under `[data-theme="dark"]`, and accept the duplication.

## Dark Mode Is Not an Inversion

- Elevation flips: in light mode raised surfaces cast shadows; in dark mode they get LIGHTER, because shadows are invisible on near-black. Ship a `--surface-raised` token, not a shadow-only story.
- Reduce chroma in dark mode. Saturated colors on dark backgrounds vibrate; drop chroma by roughly a fifth and raise lightness until contrast passes.
- Avoid pure `#000` surfaces on OLED-adjacent designs: the contrast against white text is harsh and halos on thin type. A near-black in the `#15`-`#1c` range is the common landing zone.
- Images and illustrations need attention: `filter: brightness(.9) contrast(1.05)` on photos inside dark surfaces, or ship a second asset for anything with baked-in white.
- Borders often need to become lighter overlays (`color-mix(in oklch, var(--text) 12%, transparent)`) instead of darker greys.

## Theme Flash and Switching

- CSS alone cannot read a stored preference; the theme attribute must be set before first paint by a tiny inline script in `<head>`. Anything deferred produces the white flash. The CSS side of the contract is that every themed value comes from a token so one attribute flip is enough.
- Do not transition the theme swap globally: `* { transition: background-color .3s }` repaints the entire page and janks on long documents. Either swap instantly, or transition only the few large surfaces, or use a view transition.
- Respect the OS by default: no stored preference means follow `prefers-color-scheme`, not a hardcoded light theme.
- Meta theme-color for the browser chrome is HTML, not CSS, but ship both or the address bar stays the wrong color.

## Multi-Brand and Sub-Themes

- Rebrand = swap the semantic layer only. `[data-brand="acme"] { --accent: oklch(.62 .17 250) }` and nothing else changes.
- Sub-themes on a subtree work because custom properties inherit: a dark hero inside a light page is `.hero { --surface: #101317; --text: #f2f3f5; color-scheme: dark }` — descendants pick it up with zero component changes.
- Registering tokens with `@property` gives them types, defaults, and interpolation: `@property --accent { syntax: "<color>"; inherits: true; initial-value: rebeccapurple }`. Typed tokens also stop the invalid-at-computed-value-time surprise.
- Custom properties cross the shadow DOM boundary, which makes them the only practical way to theme web components.
- Density and radius are themeable too: `--space-unit` and `--radius-md` swapped per brand cover most of what a "compact mode" needs.

## Naming That Survives

- Name by role, not by appearance: `--text-muted`, not `--grey-text`; `--danger`, not `--red`. Appearance names die the first time a theme makes them false.
- Pair every surface token with its own foreground token (`--surface` / `--on-surface`), so contrast is a property of the pair and can be checked mechanically.
- One canonical scale per axis (space, radius, type, elevation, z-index). Two scales means every component picks one arbitrarily.
- Keep a z-index scale in tokens (`--z-dropdown`, `--z-sticky`, `--z-toast`) so the numbers are comparable — though anything genuinely on top belongs in the top layer instead (SKILL.md Core Rule 7).

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Component reads a primitive token | Breaks in every theme that redefines meaning, not palette | Semantic layer only |
| Dark mode by `filter: invert()` | Inverts photos, logos, and code highlighting | Token swap |
| Only `@media (prefers-color-scheme)` | A manual toggle then cannot override the OS | `color-scheme` + `light-dark()` + attribute override |
| Shadows for elevation in dark mode | Invisible on dark surfaces; the hierarchy disappears | Lighter surface tokens |
| Theme values inlined in JS objects | Two sources of truth drift within a sprint | Tokens in CSS, read from JS via `getComputedStyle` if needed |
| Adding a token per component | The semantic layer stops being a contract and becomes a dump | Add to the semantic layer only when two components need it |
