# Components — Props, Emits, Slots, and the Built-Ins

A component is a contract: props in, events out, slots for the parts the parent owns. Most component bugs are a contract violated in one of those three directions.

## Props

```ts
const props = withDefaults(defineProps<{
  id: string
  rows?: Row[]
  variant?: 'primary' | 'ghost'
}>(), { variant: 'primary', rows: () => [] })
```

- Object and array defaults need a factory (`() => []`). A shared literal is one object shared by every instance — edit it in one place, it changes everywhere.
- Props are read-only. Mutating a passed object does work (same reference) and is still wrong: the parent has no record of the change and DevTools shows no source. Emit, or use a model.
- **Copy-once is the silent bug**: `const local = ref(props.value)` reads once and never updates. If the local copy must track the prop, `watch(() => props.value, v => local.value = v)`, or derive with `computed`.
- Boolean casting: a bare attribute is `true`, an absent one is `false`, and `disabled="false"` is the string `"false"` — which is truthy. Always bind: `:disabled="false"`.
- Static attributes are strings. `count="1"` passes `"1"`; `:count="1"` passes a number. A prop typed `number` that arrives as a string means a missing colon.
- On `vue >=3.5`, `const { rows } = defineProps<Props>()` stays reactive (the compiler rewrites accesses into getters). Below 3.5 that destructure freezes the value; use `toRef(props, 'rows')`.
- Validation beyond types belongs in a `validator` (options form) or a dev-only `watchEffect` assertion — not in a comment.

## Emits

```ts
const emit = defineEmits<{
  save: [payload: Draft]          // vue >=3.3 shorthand
  'update:open': [value: boolean]
}>()
```

- Declared emits are removed from `$attrs`; an undeclared event both fires as a listener and falls through as an attribute, which is how a `click` ends up bound twice.
- Events carry data upward; they cannot return a value. When the parent must answer (validate, veto, supply a row renderer), pass a function prop instead — an event with a callback argument is a function prop wearing a disguise.
- Name events after what happened (`row-selected`), not after what the parent should do (`open-modal`). The second name makes the component unusable in the second place it is needed.
- `emit()` returns nothing. Code branching on its return value is reading `undefined` — an event is fire-and-forget by design.

## v-model

- `v-model="x"` desugars to `:modelValue="x"` + `@update:modelValue="x = $event"`. Named models — `v-model:title` — use `title` + `update:title`.
- `defineModel()` (`vue >=3.4`) returns a ref that reads the prop and emits on write:

```ts
const title = defineModel<string>('title', { default: '' })
const count = defineModel<number>({ get: v => v ?? 0, set: v => Math.max(0, v) })
```

- Modifiers: `v-model.trim`, `.number`, `.lazy` on native inputs; on a component, `defineModel('title', { set })` plus the `titleModifiers` prop lets you implement custom ones.
- `.number` uses `parseFloat` and falls back to the raw string when parsing fails — an empty input yields `''`, not `0`. Guard before arithmetic (`forms.md`).
- A model with no parent listener still updates locally in `vue >=3.4` (the returned ref holds a local value), which makes uncontrolled usage work; do not rely on it for a controlled component's state of record.

## Slots

- Fallback content lives inside the slot tag: `<slot name="footer">Default footer</slot>`.
- Scoped slots pass data down to the content: `<slot :row="row" :index="i" />` consumed as `<template #row="{ row, index }">`.
- `$slots.footer` tests presence; call it (`$slots.footer?.()`) only when you need the rendered nodes. Testing presence with a truthiness check on the render function is reliable; inspecting its output is not.
- Dynamic slot names: `<template #[name]>`. Combine with a config-driven layout instead of a chain of `v-if`s.
- Slot content is compiled in the PARENT's scope: it can read parent state but not the child's, which is precisely why scoped slots exist.
- `defineSlots<{ row(props: { item: Item }): any }>()` (`vue >=3.3`) types them (`typescript.md`).

## Attribute Fallthrough

- A single-root component forwards unrecognized attributes (`class`, `style`, `id`, listeners) to that root automatically; `class` and `style` merge with the root's own.
- Multiple root nodes disable it and warn "Extraneous non-props attributes". Fix by choosing a target: `defineOptions({ inheritAttrs: false })` and `v-bind="$attrs"` on the element that should receive them.
- The classic case is a wrapper input: attributes must land on the inner `<input>`, not the outer `<label>` — that requires `inheritAttrs: false` even with a single root.
- `useAttrs()` reads them in script; it is not reactive per-key, so read it inside a render-tracked context.

## Dynamic and Async Components

```ts
const Editor = defineAsyncComponent({
  loader: () => import('./Editor.vue'),
  loadingComponent: Spinner,
  errorComponent: LoadFailed,
  delay: 200,        // ms before showing the loader — avoids a flash on fast networks
  timeout: 10000     // ms before switching to errorComponent
})
```

- `<component :is="X">` accepts a component object or a registered name string. Store the component in `shallowRef`, never `ref` — a deep proxy of a component definition triggers Vue's own warning and costs a full traversal.
- A name string only resolves if the component is registered; in `<script setup>` an imported component is local, so build a map (`{ text: TextCell, date: DateCell }[kind]`) instead of concatenating names.
- Async chunks are route-level wins first: splitting a 30 KB dialog matters far less than splitting a route.

## Built-In Components

| Component | Solves | Trap |
|---|---|---|
| `<KeepAlive>` | Preserving state of switched-out components | No `max` means every visited view is retained for the session; use `onActivated`/`onDeactivated`, since `onMounted` will not fire again |
| `<Teleport>` | Escaping `overflow: hidden` and z-index stacking for modals, dropdowns, toasts | The target must exist when the teleport mounts — `defer` (`vue >=3.5`) waits a tick for in-app targets; events still bubble through the Vue tree, not the DOM tree |
| `<Suspense>` | Coordinating async setup and async components | Still experimental; a single failed child resolves the whole boundary — pair with `onErrorCaptured` |
| `<Transition>` | Enter/leave of a single element | Needs one element and a stable key; `mode="out-in"` prevents both being in the DOM at once |
| `<TransitionGroup>` | List insert/remove/move | Every child needs a key, and move animations require the items to be position-animatable (`transform`, not layout) |

Transition classes in order: `v-enter-from` → `v-enter-active` → `v-enter-to`, mirrored for leave. A transition that "does nothing" is usually a missing key, a `v-show` on a `display:none` parent, or a duration Vue could not detect — set `:duration` explicitly when CSS uses nested transitions.

## Lifecycle

| Hook | Fires | Use |
|---|---|---|
| `onBeforeMount` | After setup, before first render | Rarely — no DOM yet |
| `onMounted` | After the element is in the document | DOM reads, third-party init, listeners |
| `onUpdated` | After any re-render | Diagnostics only; writing state here loops |
| `onUnmounted` | After teardown | Cleanup that setup did not register (rule 5) |
| `onActivated` / `onDeactivated` | KeepAlive enter/exit | Refetching stale data on return |
| `onErrorCaptured` | Descendant threw | Return `false` to stop propagation; the boundary pattern |

- Hooks must be registered synchronously during setup. Registered in a callback or after an `await`, they warn "called when there is no active component instance" and never run.
- `onMounted` of a child runs before the parent's; parents mount last. Measuring a child from a parent's `onMounted` is safe, the reverse is not.

## Component Design Checks

- Does the component read anything it was not given? Global store access inside a leaf component makes it untestable and unreusable — take it as a prop unless it is genuinely app state.
- Could a slot replace this prop? A `title: string` prop becomes a `#title` slot the first time someone needs a badge next to the text.
- Is `defineExpose` present? Without it, a parent's template ref on a `<script setup>` component gets an instance with nothing on it — that closure is the default, and exposing is a deliberate API decision.
