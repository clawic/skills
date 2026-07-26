# Reactivity — Refs, Watchers, and Effect Ownership

Mental model: a render or an effect that reads a reactive property subscribes to it; writing that property re-runs every subscriber. Everything below follows from that one sentence — reactivity is lost exactly when you stop holding the property and start holding a value read from it.

## ref vs reactive, Decided

| Need | Use | Why not the other |
|---|---|---|
| Primitive | `ref` | `reactive` rejects primitives outright |
| Object you will swap wholesale (API response, form reset) | `ref` | `reactive` cannot be reassigned |
| Object created once, mutated in place | either | `reactive` reads better in templates; `ref` avoids the cliffs |
| Array of thousands of rows | `shallowRef`, replaced wholesale | Deep proxying every row costs on creation and on every read |
| Class instance, DOM node, third-party handle | `markRaw` inside `shallowRef` | Deep proxy breaks `instanceof` and identity (SKILL.md rule 6) |
| Map / Set | `ref` | Collections are reactive, but ref-unwrapping does not apply inside them |
| Anything else | `ref` | One rule beats a taxonomy |

Unwrapping, precisely: a ref unwraps as a property of a `reactive` object and at the top level of a template expression. It does NOT unwrap inside arrays, inside Map/Set, or when reached through an index (`{{ list[0] }}` prints the Ref object).

## Shallow Variants

- `shallowRef(x)` tracks `.value` assignment only; mutating `x.deep.field` triggers nothing until `triggerRef(r)` or a reassignment.
- `shallowReactive(o)` tracks root-level keys only; nested objects stay raw.
- The performance case is real but bounded — reach for these on large or externally-owned data, not by default. On `vue >=3.5` the reactivity rewrite already did most of this work (the 3.5 release notes report ~56% less memory for the reactivity system and up to 10× faster operations on large deeply reactive arrays), so upgrading is usually cheaper than sprinkling `shallowRef` through a codebase.
- `triggerRef` is the escape hatch when you mutate in place for speed and then want exactly one render.

## Read-Only and Raw

- `readonly(state)` returns a proxy whose writes warn in dev and no-op — the correct return type for state exposed by a composable or provided down a tree.
- `toRaw(proxy)` returns the original object. Legitimate: handing data to a library that chokes on Proxies, or a clone/serialization boundary. Illegitimate: mutating it — those writes are invisible to Vue.
- `markRaw(obj)` opts an object out of reactivity permanently, including when nested in reactive state. It is shallow: a proxy created earlier for a child is unaffected.
- `Object.freeze(data)` before wrapping makes the result non-reactive with no warning at all. Freeze after rendering, or choose `shallowRef` deliberately.

## Watchers

```js
watch(source, cb, { immediate, deep, flush, once })
```

- **Source shapes**: a ref, a getter `() => a.b`, a reactive object (forced deep), or an array of these. A plain value is invalid and warns "Invalid watch source".
- **`immediate: true`** runs once at setup with `oldValue === undefined` — that is how you detect the first run, not a hand-rolled flag.
- **`deep: true`** traverses the whole tree on every mutation; cost scales with node count. On `vue >=3.5`, `deep: 2` bounds the traversal. Watching a getter of the exact field beats both.
- **`once: true`** (`vue >=3.4`) self-stops after the first call, replacing `const stop = watch(...)` plus `stop()` inside the callback.
- **Object identity**: with a reactive source or `deep`, `newValue` and `oldValue` are the same reference. Snapshot in the getter (`() => structuredClone(toRaw(o))`) if you truly need the previous state.
- **`watchEffect`** tracks whatever it reads on its first run, runs immediately, gives no old value, and re-tracks on every run — a branch not taken on run 1 is not tracked. Use `watch` when you need laziness, old values, or a source that is not read in the body.

## Flush Timing

| Flush | Runs | Use for |
|---|---|---|
| `pre` (default) | Before the component re-renders, batched per tick | Deriving state, validation, syncing to a store |
| `post` | After the DOM patch | Measuring elements, focusing inputs, feeding a chart library |
| `sync` | Synchronously on every mutation, unbatched | Almost never — a loop with 1000 writes fires 1000 callbacks |

`watchPostEffect` and `watchSyncEffect` are the `watchEffect` equivalents. `await nextTick()` inside a `pre` watcher reaches the same DOM as `post`, with more ceremony.

## Cleanup and the Async Race

```js
watch(id, async (newId, oldId, onCleanup) => {
  const ctrl = new AbortController()
  onCleanup(() => ctrl.abort())            // before the next run AND on unmount
  user.value = await fetchUser(newId, { signal: ctrl.signal })
})
```

- Without the abort, a slow response for `oldId` lands after a fast one for `newId` and overwrites it — the race that makes a profile page display the previously selected user.
- `onWatcherCleanup(fn)` (`vue >=3.5`) does the same from anywhere in the callback, but must be called synchronously, before the first `await`.
- `watchEffect((onCleanup) => ...)` uses the same mechanism.

## Effect Scope Ownership

Effects registered synchronously during setup belong to the component and stop at unmount. Everything else is yours:

```js
const scope = effectScope()
scope.run(() => {
  watch(source, cb)                      // owned by this scope
  watchEffect(update)
  onScopeDispose(() => socket.close())   // runs on scope.stop()
})
scope.stop()                             // stops every effect created inside
```

- Use it for a composable that creates effects lazily (after an `await`, in a timer, in a store action) and for singletons that outlive any component.
- `getCurrentScope()` tells you whether cleanup is already handled — the guard a library-grade composable checks before registering `onScopeDispose`.
- The boundary is the synchronous call stack, not the lifecycle hook: a watcher created inside `onMounted` is owned; the same watcher created inside a `setTimeout` is not.
- `scope.stop()` is idempotent; `effectScope(true)` (detached) refuses to be collected by a parent scope, which is what you want for a module-level singleton.

## Computed

- Cached: it re-evaluates only when a tracked dependency changed AND someone reads it. A computed nobody reads never runs.
- Writable form, for two-way derived state:

```js
const fullName = computed({
  get: () => `${first.value} ${last.value}`,
  set: (v) => { [first.value, last.value] = v.split(' ') }
})
```

- Getters must be pure and synchronous. `async` in a computed returns a Promise, tracks nothing past the first `await`, and renders `[object Promise]`.
- Chains are cheap: five small computeds each cache independently and beat one large one that recomputes whenever any input changes.
- `onTrack` / `onTrigger` options (dev builds only) name the exact dependency that fired — faster than bisecting by comment.

## Crossing Boundaries

- **Into a composable**: accept `MaybeRefOrGetter` and normalize with `toValue(x)` (`vue >=3.3`); value, ref, and getter then all work from one signature.
- **Out of a composable**: return refs (destructurable), not a `reactive` object (not destructurable).
- **Into a template**: refs unwrap, getters do not — `{{ myGetter() }}` needs the call.
- **Into a non-Vue library**: `toRaw()` at the boundary, and re-push later mutations yourself; the library will not observe them.
- **Into a store**: values and refs, never a component instance or a DOM node.
