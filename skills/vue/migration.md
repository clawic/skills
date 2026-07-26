# Migration — Vue 2 to Vue 3, Options to Composition

Two migrations that are often confused and should be sequenced: Vue 2 → Vue 3 is a dependency and API migration with hard breaks; Options API → Composition API is a style migration with none. Do the first for support, the second only where it pays.

Vue 2 reached end of life on 2023-12-31: no patches, including security ones. That is the deadline argument; everything below is the mechanics.

## Sequencing

1. **Upgrade to Vue 2.7 first.** It backports the Composition API, `<script setup>`, and the newer tooling to Vue 2 — code written against it moves to Vue 3 with far fewer edits.
2. **Inventory the ecosystem.** Vue Router 3→4, Vuex 3→4 (or Pinia), the component library, and every plugin that touches internals. A UI library with no Vue 3 release is the whole project's blocker, and finding it in week one changes the plan.
3. **Run the migration build (`@vue/compat`)** to get a running app with warnings instead of a big-bang rewrite. Silence warnings feature by feature, then remove the compat layer.
4. **Convert to Composition API afterward, selectively.** Mixing the two APIs is supported indefinitely; a working Options component is not debt.

## Hard Breaks

| Vue 2 | Vue 3 | Migration |
|---|---|---|
| `new Vue({...})` | `createApp(App)` | Global config moves onto the app instance; multiple apps no longer share config |
| `Vue.prototype.$http` | `app.config.globalProperties.$http` | Better: `provide`/`inject` or a plain import |
| `Vue.use`, `Vue.mixin`, `Vue.component` globally | `app.use`, `app.mixin`, `app.component` | Anything registered before an app existed must move into a setup function |
| Filters (`{{ x \| currency }}`) | Removed | A method or a computed |
| `$on` / `$off` / `$once` event bus | Removed | Store, provide/inject, or an external emitter library |
| `.sync` modifier | `v-model:arg` | `:title.sync="t"` → `v-model:title="t"` |
| `v-model` = `value` + `input` | `modelValue` + `update:modelValue` | Component-by-component; `defineModel()` on `vue >=3.4` |
| `Vue.set` / `Vue.delete` | Removed | Proxy reactivity tracks property addition and deletion natively |
| `beforeDestroy` / `destroyed` | `beforeUnmount` / `unmounted` | Mechanical rename |
| `$listeners` | Merged into `$attrs` | Fallthrough now includes listeners (`components.md`) |
| `$children` | Removed | Template refs, or restructure so the parent does not reach in |
| `functional: true` / functional SFCs | Removed | Plain components; the performance gap closed |
| `v-if` lower priority than `v-for` | `v-if` now evaluates first | Filter upstream or split with `<template>` (`templates.md`) |
| `key` on `<template v-for>` children | `key` on the `<template>` | Move it up one element |
| Single root required | Fragments allowed | Multi-root disables attribute fallthrough — a silent behavior change (`components.md`) |
| `$scopedSlots` | Unified into `$slots` (all slots are functions) | Call them: `$slots.default?.()` |
| `render(h)` receives `h` | Import `h` from `vue` | Also: children as arrays vs slots as objects |
| Async component as a bare function | `defineAsyncComponent(() => import(...))` | Wrap it |
| `transition` class names `v-enter`/`v-leave-to` | `v-enter-from`/`v-leave-to` | Rename the CSS |

## Reactivity Differences That Change Behavior

- Vue 3 uses Proxies: adding and deleting properties is tracked, array index assignment works, and `Vue.set` is unnecessary. Code that carefully avoided those patterns can be simplified.
- Proxies cannot track a value you destructured — a limitation Vue 2's `Object.defineProperty` shared, but Vue 3's Composition API makes it far easier to hit (SKILL.md Reactivity Loss).
- A Proxy is not the raw object: `instanceof` against a class stored in reactive state, and `===` comparisons with the original, now fail. This is the most common runtime surprise for ported code (SKILL.md rule 6).
- `data()` is now always a function, and the root instance has no `$children` — code that walked the instance tree needs a different design, not a translation.

## Options → Composition, Mechanically

| Options | Composition |
|---|---|
| `data() { return { a: 1 } }` | `const a = ref(1)` |
| `computed: { b() {} }` | `const b = computed(() => ...)` |
| `watch: { a: 'handler' }` | `watch(a, handler)` |
| `methods: { m() {} }` | `function m() {}` |
| `mounted()` | `onMounted(() => {})` |
| `props: {...}` | `defineProps<{...}>()` |
| `this.$emit('x')` | `const emit = defineEmits<...>()` then `emit('x')` |
| `this.$refs.el` | `useTemplateRef('el')` (`vue >=3.5`) or `const el = ref(null)` |
| `mixins: [A, B]` | Composables — the actual reason to migrate |
| `provide/inject` options | `provide()` / `inject()` in setup |

- There is no `this` in setup. A method that reached across `this.$parent`, `this.$root`, or `this.$refs.child.method()` needs a redesign, not a rewrite.
- Mixins are the migration's real payoff: two mixins with the same property name silently overwrote each other, and neither declared its dependencies. Composables have explicit inputs and outputs.
- Options components inside a Composition app work unchanged. Convert a file when you are already editing it for another reason.

## What Not to Migrate

- A stable Options API screen that nobody touches: converting it introduces risk and buys nothing measurable.
- Vuex to Pinia mid-migration: do the Vue 3 upgrade on Vuex 4 (which runs on Vue 3), then move stores in a second pass. Two migrations at once means every bug has two suspects.
- A component that only exists because a mixin needed a host — delete it instead.

## Verification per Batch

- The migration build's warnings go to zero for the files in the batch before the batch is called done.
- `vue-tsc --noEmit` after each batch (`typescript.md`); the type errors reveal the props and emits contracts that changed shape.
- Manually exercise attribute fallthrough on every converted wrapper component — multi-root fragments break it silently and no test written against Vue 2 covers it.
- Grep the batch for `$refs`, `$parent`, `$children`, `$listeners`, and `filters:` before declaring it converted.
