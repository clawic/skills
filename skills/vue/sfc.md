# SFC — script setup, Scoped CSS, and Compiler Behavior

The `.vue` file is three compiled languages in one. Knowing which block runs when — and how the style block is rewritten — resolves most "why does this not apply" questions.

## Block Semantics

| Block | Runs | Notes |
|---|---|---|
| `<script setup>` | Per component instance, as the setup function | Top-level bindings are exposed to the template; nothing is exposed to a parent unless `defineExpose` |
| `<script>` (plain) | Once, at module evaluation | For code that must run once, or options the macros cannot express; can coexist with `<script setup>` |
| `<template>` | Compiled to a render function at build time | Only setup bindings and globals are in scope |
| `<style>` | Extracted at build time | `scoped`, `module`, and `src` are the three modes |
| Custom blocks (`<i18n>`, `<docs>`) | Whatever a plugin does with them | Requires a build plugin; unresolvable in a plain runtime |

- Both script blocks share a module scope, so a `const` in the plain block is visible in the setup block. Only setup bindings reach the template.
- `src="./logic.ts"` on a block imports its content — the one supported way to split an SFC without losing tooling.

## Compiler Macros

| Macro | Since | Purpose |
|---|---|---|
| `defineProps` / `defineEmits` | 3.2 | The component's contract |
| `defineExpose` | 3.2 | Opts specific bindings into the parent's template ref |
| `defineOptions` | 3.3 | `name`, `inheritAttrs`, and other options without a second script block |
| `defineSlots` | 3.3 | Type-only slot declaration |
| `defineModel` | 3.4 | Two-way binding as a ref |
| `useTemplateRef` / `useId` | 3.5 | String-keyed template refs; SSR-stable ids |

- `vue >=3.2` is the floor for the whole syntax: `<script setup>` shipped stable in 3.2 alongside `defineExpose`. In 3.0/3.1 it was an experimental RFC with different spellings (`defineEmit`, singular; expose via `useContext()`), so version-gate anything you write for a codebase below 3.2 by upgrading it, not by porting the old spellings.
- Macros are compile-time only: they must appear at the top level of `<script setup>`, cannot be imported, cannot be called conditionally, and cannot reference variables declared later in the same block.
- They compile away entirely. A macro that does not exist in the installed version is just an undefined function call at runtime (SKILL.md rule 8).
- `defineProps` type arguments accept imported and complex types from `vue >=3.3`; earlier versions require the type to be declared inline in the same file.

## Generic Components

```vue
<script setup lang="ts" generic="T extends { id: string }">
defineProps<{ items: T[]; selected?: T }>()
defineEmits<{ select: [item: T] }>()
</script>
```

- `vue >=3.3`. The type parameter flows through props, emits, and slots, so a `<DataTable>` returns the caller's row type instead of `any`.
- Constrain the parameter (`extends`) or the component accepts anything and the typing buys nothing.

## Scoped CSS

`<style scoped>` adds a `data-v-xxxxxx` attribute to the component's own elements and rewrites every selector to require it. Consequences, in order of how often they bite:

- Child components' internal elements do not match — that is the point. Reach in deliberately with `:deep(.child-class)`, which compiles to `[data-v-x] .child-class`.
- The child's ROOT element does match the parent's scope (it carries both attributes), so styling a child's root needs no `:deep`.
- Slot content is compiled in the parent, so it carries the PARENT's scope id. Style it from the child with `:slotted(.item)`.
- `:global(.selector)` escapes scoping for a single rule — for a body class or a third-party widget.
- `>>>`, `/deep/`, and `::v-deep` are the deprecated spellings of `:deep()`; some preprocessors also choke on them.
- Scoping is not isolation: a bare element selector (`div { padding: 0 }`) still applies to every div in this component including children's roots, and specificity fights with global CSS the same as always.
- Dynamic values in CSS: `v-bind(color)` inside `<style>` (`vue >=3.2`) compiles to a CSS custom property updated reactively — cheaper than binding `:style` on every element.

```vue
<style scoped>
.card { color: v-bind(textColor); }
.card :deep(.icon) { width: 1rem; }
:slotted(.tag) { margin-inline-end: .5rem; }
</style>
```

## CSS Modules

```vue
<style module>
.title { font-weight: 600 }
</style>
<template><h2 :class="$style.title" /></template>
```

- `<style module>` exposes `$style` (or a named object with `module="cls"`); `useCssModule()` reads it in script.
- Class names are hashed, so collisions are impossible — stronger than scoped, at the cost of `$style.` everywhere and no styling of children at all.
- Scoped and module can coexist in one file; do not use both for the same concern.

## Compiler Details That Surface as Bugs

- Component name resolution in templates: `<MyThing>` matches an import named `MyThing`; kebab-case in the template resolves to the PascalCase import. In-DOM templates (a `<div id="app">` in HTML) are lowercased by the browser and must use kebab-case.
- The template is not HTML. Self-closing custom tags (`<MyThing />`) work in SFCs and fail in in-DOM templates.
- Whitespace handling condenses runs between elements; `whitespace: 'preserve'` in the compiler options changes it if a layout depends on the space between inline elements.
- Expressions in templates are restricted to a single expression — no statements, no `var`. Multi-step logic goes into a computed, which is also where it becomes testable.
- HMR preserves state for `<script setup>` components but recreates them when the props signature changes; a stale state after an edit is usually HMR, not your code. Confirm with a full reload before debugging.

## SFC Review Checks

- Every macro used exists in the project's Vue version?
- No `:deep()` used where the child's root element would already match?
- Slot content styled with `:slotted`, not with a `:deep` guess?
- Component `name` set with `defineOptions` if `KeepAlive :include` or recursive self-reference depends on it?
- No business logic in the template expression that a computed should own?
