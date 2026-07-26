# TypeScript — Typing Components, Refs, and Injections

Vue's types are good enough that most annotations are unnecessary; the work is at the boundaries — props, emits, slots, template refs, injections, and stores. Language-level type design (conditional types, generics, declaration files) is `typescript`.

## Toolchain

- `tsc` does not understand `.vue` files. Type-check SFCs with `vue-tsc --noEmit` and wire it into CI — the editor's checks are per-file and miss cross-component breakage.
- The bundler transpiles without type-checking (esbuild/SWC strip types), so a broken build passes `dev` and fails only in CI. That gap is by design and the reason the check must be a separate script.
- `"strict": true` plus `"noUncheckedIndexedAccess"` catches the `list[0]` case that produces most runtime `undefined` errors in table code.
- `env.d.ts` needs `/// <reference types="vite/client" />` for `import.meta.env`; the `.vue` module shim ships with the Vue plugin, so hand-written shims are usually a leftover that hides real errors.

## Props and Emits

```ts
interface Props {
  items: Row[]
  selected?: string
  variant?: 'primary' | 'ghost'
}
const props = withDefaults(defineProps<Props>(), { variant: 'primary' })

const emit = defineEmits<{
  select: [id: string]
  'update:open': [open: boolean]
}>()
```

- Type-only declaration (`defineProps<T>()`) and runtime declaration (`defineProps({...})`) are mutually exclusive — combining them is a compile error.
- Imported and complex types work in the type argument from `vue >=3.3`; earlier versions require a locally declared interface.
- `withDefaults` is unnecessary on `vue >=3.5` if you destructure: `const { variant = 'primary' } = defineProps<Props>()` keeps reactivity and reads better.
- The tuple-syntax emits (`select: [id: string]`) is `vue >=3.3`; the call-signature form (`(e: 'select', id: string): void`) works everywhere and is what you need on 3.2.
- Optional props are `T | undefined` in the component body even with a default, unless you use `withDefaults` or the destructure form — the type reflects what the parent may pass.

## Template Refs

```ts
const el = useTemplateRef<HTMLInputElement>('input')      // vue >=3.5
const child = useTemplateRef<InstanceType<typeof ChildComp>>('child')

const legacy = ref<HTMLInputElement | null>(null)         // any version, ref="legacy"
```

- The `| null` is not pedantry: the ref is null before mount and whenever the element is behind a false `v-if`.
- `InstanceType<typeof Comp>` types a child component ref — but only `defineExpose`d members are on it for a `<script setup>` component.
- For a generic child, `InstanceType` collapses the type parameter; type the exposed surface as an explicit interface and cast to it.

## Slots and Attrs

```ts
defineSlots<{
  default(props: { row: Row; index: number }): any
  empty?(): any
}>()
```

- `vue >=3.3`. It types the child's slot contract and gives the parent autocomplete inside `<template #default="{ row }">`.
- `useAttrs()` returns `Record<string, unknown>` — narrow at the point of use rather than casting the whole object.

## provide / inject

```ts
// keys.ts — one module, so provider and consumer cannot drift
export const CartKey = Symbol('cart') as InjectionKey<{
  items: Readonly<Ref<Line[]>>
  add: (l: Line) => void
}>

provide(CartKey, { items: readonly(items), add })
const cart = inject(CartKey)          // typed, and possibly undefined
const cart2 = inject(CartKey)!        // assert only where a provider is guaranteed
```

- Without an `InjectionKey`, `inject('cart')` is `unknown` and every consumer casts — the casts drift from the provider silently.
- Prefer a local wrapper composable (`useCart()` that injects and throws a named error when absent) over `!` at every call site; the error message pays for itself the first time someone renders the consumer outside the provider.

## Reactive Types

| Value | Type |
|---|---|
| `ref(0)` | `Ref<number>` |
| `ref<User \| null>(null)` | `Ref<User \| null>` — annotate when the initial value is narrower than reality |
| `computed(() => x.value * 2)` | `ComputedRef<number>` (read-only) |
| Writable computed | `WritableComputedRef<T>` |
| `reactive(obj)` | `UnwrapNestedRefs<T>` — nested refs are unwrapped in the type too |
| Composable input | `MaybeRefOrGetter<T>`, read with `toValue` |
| `shallowRef(x)` | `ShallowRef<T>` — assignment to `.value` is not deeply typed as reactive |

- `ref<Row[]>([])` beats `ref([])`, which infers `Ref<never[]>` and rejects every push.
- Do not annotate what is inferred correctly; a wrong annotation on a computed silences a real error.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Cannot find module './X.vue' or its corresponding type declarations` | Missing plugin types, or `vue-tsc` not run | Ensure the Vue plugin's `.d.ts` is included by `tsconfig` |
| `Type 'Ref<T>' is not assignable to 'T'` | Passing a ref where the value belongs | `.value`, or `toValue()` in a composable |
| `Property 'x' does not exist on type 'ComponentPublicInstance'` | Reading a non-exposed member off a child ref | `defineExpose` it |
| `Type instantiation is excessively deep` | A recursive generic, often from a large schema library | Break the type with an explicit annotation at the boundary |
| Props type "not assignable" only in the parent template | The child's prop type narrowed (a literal union) while the parent passes `string` | Type the parent's value as the union, not as `string` |
| Emits typed but the parent's handler is `any` | Emits declared with the runtime array form | Switch to the type-only form |

## Typing Review Checks

- Is `vue-tsc --noEmit` part of the build or CI, not just the editor?
- Do injections use `InjectionKey` from a shared keys module?
- Are template refs typed and null-checked?
- Is every `ref([])` and `ref(null)` annotated with the eventual type?
- Are exposed members of `<script setup>` children declared with `defineExpose`, so their type exists?
