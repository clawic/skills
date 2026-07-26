# Debugging — Symptom to Cause in Minutes

Work symptom-first. Each chain is ordered by probability, and every step is a check you can run, not a guess. Warning strings decode via the Console Warnings table in SKILL.md.

## The Universal First Three

1. **Is the state actually changing?** `watch(source, v => console.log('src', v), { immediate: true })` next to the value. Silence means the mutation never happened or you are watching a snapshot (SKILL.md Reactivity Loss). Noise with a static DOM means the render is the problem.
2. **Is the component the one you think?** Vue DevTools → select the element → confirm the instance, its props, and its `setup` state. Duplicated component names and a stale `<KeepAlive>` cache both produce "my edits do nothing".
3. **Does it survive isolation?** Copy the component into a blank route with literal props and no store. If it works there, the bug is an input; if it fails there, the bug is inside.

## UI Does Not Update

1. Log the source, not the derived value — a computed that never recomputes points at a dependency that was never tracked.
2. Check the ownership of the value: destructured? `toRaw`d? frozen? read from a prop into a local `ref` once? → SKILL.md Reactivity Loss.
3. Mutation through a non-reactive path: `arr[i] = x` and `arr.length = 0` are tracked in Vue 3, but writes to an object obtained from `toRaw()`, from a Map stored in `shallowReactive`, or to a class instance under `markRaw` are not.
4. It renders once and then freezes → `v-once`, `v-memo` with a dependency list missing the value, or a `shallowRef` mutated in depth (`triggerRef` forces it).
5. Update happens but paints late → you are reading the DOM in a `pre` watcher (SKILL.md rule 4).
6. Update happens for one instance only → the component is keyed by index and Vue reused the wrong instance (SKILL.md rule 7).

## Infinite Loop ("Maximum recursive updates exceeded")

The message means one render scheduled another, ~100 times in a row. Find the cycle:

1. Comment out watchers one at a time — the loop almost always has a watcher in it: it reads A and writes B, while another effect reads B and writes A.
2. Look for a watcher writing to its own source: `watch(list, () => list.value.sort())` mutates what it observes.
3. Look inside `computed` for a mutation (push, sort, assignment) — a computed that writes is a loop generator, not a derivation.
4. A template that calls a function which mutates state during render: `{{ formatAndCache(item) }}` where the function writes to a ref.
5. `deep: true` plus normalizing the watched object inside the callback: normalization counts as a change.
6. Fix by breaking the cycle, not by adding a flag: derive one side with `computed`, or guard the write with a value comparison (`if (next !== current)`).

## Memory Grows Over Navigation

Symptom: the tab's heap climbs every time the user visits the same route.

1. Take a heap snapshot, navigate away and back three times, snapshot again, compare retained detached DOM nodes.
2. Listeners added in `onMounted` (`window`, `document`, `IntersectionObserver`, `setInterval`, WebSocket, third-party widgets) with no `onUnmounted` counterpart — the most common cause by a wide margin.
3. Effects created after an `await` in setup, or inside a callback: not owned by the component scope (SKILL.md rule 5). Grep for `watch(` and `watchEffect(` inside `then`, `async` bodies, and event handlers.
4. `<KeepAlive>` with no `max` retains every visited instance for the session, by design — that is not a leak, but it looks like one.
5. A store holding component instances, DOM nodes, or a large `results` array that is appended and never trimmed.
6. Global event bus or plugin registering per component and never unregistering.

## "It Works in Dev, Breaks in Production"

| Difference | Check |
|---|---|
| Warnings stripped in prod builds | Rebuild in dev mode to see the warning the prod build swallowed |
| Runtime compiler absent in prod build | String `template:` options and `v-html`-injected templates fail — "runtime compilation is not supported" |
| Tree-shaking removed a dynamic reference | Component resolved by a runtime string name (`:is="'My' + kind"`) has no static reference to keep |
| Minified component names | `Failed to resolve component` in prod only → registration relied on `name` inference from filename |
| Env vars | Only `VITE_`-prefixed vars reach client code; others are `undefined` at runtime, not at build time |
| Source order / CSS specificity | Scoped styles win differently once minified and concatenated (`sfc.md`) |
| Base path | Assets 404 under a sub-path deploy; router history mode 404s on refresh without a server fallback (`routing.md`) |

## Hydration Mismatch

Full chain in `ssr.md`. The 60-second version: the server HTML and the first client render differ. Read the warning's node path, then check — non-deterministic values (`Date`, `Math.random`, `crypto`), browser-only reads (`window`, `localStorage`, `navigator`) during setup, invalid HTML nesting the browser silently repairs (`<div>` inside `<p>`, rows outside `<tbody>`), locale or timezone differences between server and client, and browser extensions injecting nodes.

## Template Ref Is Null

1. Reading during setup — refs bind at mount. Move into `onMounted`, or watch the ref.
2. Element is behind `v-if` and currently false — the ref is null by definition; watch it instead of reading once.
3. Name mismatch between `ref="x"` and the variable — exact string. On `vue >=3.5`, `useTemplateRef('x')` removes the class of bug where the variable was renamed.
4. `v-for` refs collect into an array whose order is not guaranteed to match the source list — index into it only after keying by identity yourself.
5. The ref points at a component, not an element: you get the instance, and `<script setup>` closes it unless it calls `defineExpose`.

## Props Arrive Wrong

1. `undefined` where you expected a value → attribute vs prop casing: `my-prop` in the template maps to `myProp`; a hyphenated name in a JSX/render function does not.
2. Everything is a string → static attributes are strings; bind with `:` to pass numbers, booleans, arrays.
3. Boolean prop is `true` when omitted → an empty attribute means `true`; use `:disabled="false"`, not `disabled="false"`.
4. Object default resets or is shared between instances → object and array defaults need a factory (`default: () => ({})`).
5. Prop updates but the child does not react → the child copied it into a local `ref` in setup, once (`components.md`).
6. Extra attributes land nowhere → multi-root component; fallthrough needs a single root or explicit `$attrs`.

## Store Behaves Oddly

1. `"getActivePinia()" was called but there was no active Pinia` (older Pinia: `getActivePinia was called with no active Pinia`) → the store was called at module scope or before `app.use(pinia)`; call it inside setup.
2. Destructured state is frozen → `storeToRefs` (SKILL.md Reactivity Loss).
3. State bleeds between users in SSR → one Pinia instance shared across requests (`ssr.md`).
4. `$reset()` throws → only options stores have it; setup stores need one written by hand (`state.md`).
5. Two components see different data → two Pinia instances, usually a duplicated `vue` or `pinia` copy in the dependency tree; check for duplicate packages before suspecting your code.

## Build and Type Errors

- `Cannot find module './X.vue' or its corresponding type declarations` → missing shim or `vue-tsc` not in the pipeline (`typescript.md`).
- `[plugin:vite:vue] ...` at a line that looks fine → an unclosed tag or a stray `<` in an earlier template block; the compiler reports where it gave up, not where you erred.
- Macro is `undefined` at runtime → the macro postdates the installed Vue version (SKILL.md rule 8), or it was called conditionally; compiler macros must appear at the top level of `<script setup>`.
- Two `<script>` blocks disagreeing → a plain `<script>` and `<script setup>` can coexist, but only the setup block's bindings reach the template.

## When You Are Truly Stuck

Bisect the component tree: replace the child's template with a single static `<div>`. Working means the bug is below; still broken means it is above. Repeat — five bisections cover a tree of 32 components, and each step names the next file above to open.
