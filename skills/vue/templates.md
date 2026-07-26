# Templates — Directives, Rendering, and Custom Directives

The template compiles to a render function; every directive is a transform with rules about ordering and scope. The bugs here come from assuming the template is HTML.

## Directive Priority

Vue 3 evaluates in this order on one element: `v-if` → `v-for` → everything else. Vue 2 was the reverse, which is why ported code breaks.

```vue
<!-- Broken in Vue 3: v-if runs first and cannot see `item` -->
<li v-for="item in items" v-if="item.active" :key="item.id">

<!-- Correct: filter upstream (preferred — no wasted nodes) -->
<li v-for="item in activeItems" :key="item.id">

<!-- Or split the concerns -->
<template v-for="item in items" :key="item.id">
  <li v-if="item.active">{{ item.label }}</li>
</template>
```

- The key goes on `<template v-for>`, not on the inner element (that moved in Vue 3).
- `v-for` over an object iterates `(value, key, index)`; over a number it starts at 1, not 0.
- `v-for` and `v-once` together freeze the list at first render — a real technique for static tables, an accidental bug otherwise.

## v-if vs v-show

| | `v-if` | `v-show` |
|---|---|---|
| Absent branch | Not created; children never mount | Created and mounted, `display: none` |
| Toggle cost | Mount + unmount each time | One style change |
| Initial cost | Zero when false | Full render even when hidden |
| Works on `<template>` | Yes | No |
| Lifecycle hooks | Fire on every toggle | Fire once |

Rule: `v-show` for something toggled frequently and cheap to build; `v-if` for something rarely shown, expensive, or that must not run its setup (a modal that fetches on mount, an editor that grabs focus).

## Bindings

- `:class` accepts a string, an object (`{ active: isActive }`), or an array mixing both. Object and array forms merge with a static `class` on the same element.
- `:style` accepts camelCase or kebab-case keys and an array of objects; an array of values for one property (`display: ['-webkit-box', 'flex']`) renders the last one the browser supports.
- `v-bind="obj"` spreads an object of attributes; later bindings win, so put the spread first when a specific binding must override it.
- Dynamic argument: `:[attrName]="value"` and `@[eventName]="handler"`. The expression must resolve to a lowercase string or `null` (which removes the binding) — in-DOM templates lowercase attribute names, which breaks camelCase dynamic args.
- Same-name shorthand `:id` (`vue >=3.4`) binds `id` to the `id` in scope.

## Event Modifiers

| Modifier | Effect | Note |
|---|---|---|
| `.stop` / `.prevent` | `stopPropagation()` / `preventDefault()` | Order matters: `.prevent.self` prevents everything, `.self.prevent` only when the target is the element |
| `.self` | Only when `event.target` is this element | The correct way to close a modal by clicking its backdrop |
| `.capture` | Capture phase | For intercepting before children |
| `.once` | Auto-removes after one call | Works on component events too |
| `.passive` | Never calls `preventDefault` | For `scroll`/`touchmove`; cannot combine with `.prevent` |
| `.exact` | Only with exactly those system keys | `@click.ctrl` fires on Ctrl+Shift+click; `@click.ctrl.exact` does not |
| Key modifiers | `.enter`, `.esc`, `.tab`, `.delete`, `.space`, plus any kebab-case `event.key` | `@keyup.page-down` works because it kebab-cases `PageDown` |

Modifiers are compile-time transforms and apply to native events; on a component, `@click` listens for an emitted `click` unless the component forwards attributes to a DOM node.

## Rendering Text and HTML

- `{{ }}` escapes. It is the only interpolation that is safe with user data.
- `v-text` sets `textContent` (equivalent, useful to avoid the flash of un-compiled mustaches in in-DOM templates); `v-html` sets `innerHTML` and is an XSS hole for anything a user can influence (`security.md`).
- `v-html` content is NOT compiled as a Vue template: directives and components inside it are inert markup.
- `v-pre` skips compilation of a subtree — the way to show literal `{{ }}` in documentation.
- `v-cloak` plus a CSS rule hides un-compiled markup; only relevant for in-DOM templates without a build step.

## Custom Directives

```ts
const vFocusTrap: Directive<HTMLElement, boolean> = {
  mounted(el, binding) { if (binding.value) trap(el) },
  updated(el, binding) {
    if (binding.value === binding.oldValue) return   // guard: updated fires on every parent render
    binding.value ? trap(el) : release(el)
  },
  unmounted(el) { release(el) }
}
```

- Hooks: `created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`. A function shorthand registers `mounted` + `updated` only — no cleanup, which is how directive-based listeners leak.
- `binding` carries `value`, `oldValue`, `arg`, `modifiers`, and `instance`. Compare `value` to `oldValue` before doing work; `updated` fires whenever the owning component re-renders, value change or not.
- On a component, a custom directive applies to the single root element and warns for multi-root components — one more reason for a single root.
- In `<script setup>`, a `const vName` is auto-registered as `v-name`. The `v` prefix in the variable name is required.
- Reach for a directive only for direct DOM behavior (focus, tooltips, intersection, click-outside, autoresize). Anything with state or markup is a component.

## Render Functions and JSX

- `h(type, props, children)` is the escape hatch for structure computed at runtime: a table whose column components come from config, a recursive tree, a component that returns one of six shapes.
- Children as an array are element children; as an object they are slots (`{ default: () => [...] }`) — passing slots as an array is the usual first mistake.
- Templates are faster by default: the compiler emits patch flags and hoists statics that a hand-written render function cannot express. Do not convert templates to `h()` for performance.
- JSX in Vue is not React's: no `className`, `v-model` needs the plugin's transform, and event props are `onClick`.

## Template Review Checks

- No `v-if` on a `v-for` element?
- Every `v-for` keyed by identity?
- No user-influenced string reaching `v-html`?
- Frequently toggled branches on `v-show`, expensive rare ones on `v-if`?
- Custom directives comparing `value` to `oldValue` and cleaning up in `unmounted`?
