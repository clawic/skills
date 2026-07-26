# Composables — Reusable Logic That Cleans Up After Itself

A composable is a function that owns reactive state and its effects. The quality bar is not "it has a `use` prefix" — it is: flexible input, ref-shaped output, effects that stop when the caller disappears, and safety when it is called on a server.

## The Canonical Shape

```ts
export function useUser(id: MaybeRefOrGetter<string>) {
  const data = ref<User | null>(null)
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  watch(() => toValue(id), async (value, _old, onCleanup) => {
    if (!value) return
    const ctrl = new AbortController()
    onCleanup(() => ctrl.abort())
    isLoading.value = true; error.value = null
    try { data.value = await fetchUser(value, { signal: ctrl.signal }) }
    catch (e) { if (!ctrl.signal.aborted) error.value = e as Error }
    finally { isLoading.value = false }
  }, { immediate: true })

  return { data: readonly(data), error: readonly(error), isLoading, refresh: () => {/* … */} }
}
```

Four decisions encoded above: `MaybeRefOrGetter` input, abort on re-run, `isLoading` reset in `finally` (not after the await, which a throw skips), and `readonly` on state the caller must not write.

## Input Contract

- Accept `MaybeRefOrGetter<T>` and unwrap with `toValue()` (`vue >=3.3`). One signature then serves `useX('a')`, `useX(someRef)`, and `useX(() => props.a)`.
- Never accept the props object and destructure it inside — that is the reactivity loss from SKILL.md, moved one file away where it is harder to see.
- Options go in a single object argument with defaults, so adding the fourth option is not a breaking change.
- Reject invalid input by returning an inert result (empty refs plus an `error`), not by throwing during setup — a throw in setup unmounts the whole subtree.

## Output Contract

- Return an object of refs so the caller destructures what it needs. Returning `reactive({...})` forces the caller to keep the wrapper or lose reactivity.
- Return `readonly()` state plus explicit mutators. It costs one wrapper and removes an entire category of "who wrote this value" bugs.
- Return the stop handle for anything long-lived (`stop`, `pause`, `resume`) — a composable that starts a poller with no way to stop it is a leak with an API.
- Keep the shape stable across states: `{ data, error, isLoading }` always present, never `data | undefined` on the object itself.

## Lifecycle and Ownership

- Lifecycle hooks inside a composable attach to the component that called it — but only if the call is synchronous during setup. Called from an event handler or after an `await`, they warn and no-op.
- For lazy or out-of-component use, own an `effectScope()` and expose `stop()` (`reactivity.md`).
- Guard optional cleanup: `if (getCurrentScope()) onScopeDispose(cleanup)`. Without the guard, a composable used in a plain module warns on every call.
- Each call creates a fresh instance. Shared state requires module-scope refs — which is a global store: one instance for every component, and on a server one instance for every concurrent request, so one user's data renders into another's page (`ssr.md`).

```ts
// Shared vs per-call, made explicit
const count = ref(0)                       // module scope: ONE instance, app-wide, leaks across SSR requests
export function useCounter() { return { count } }

export function useLocalCounter() {        // per-call: fresh state each time
  const count = ref(0); return { count }
}
```

## Browser APIs Safely

```ts
export function useMediaQuery(query: string) {
  const matches = ref(false)
  if (typeof window === 'undefined') return matches   // SSR: inert, no crash
  const mql = window.matchMedia(query)
  const update = () => (matches.value = mql.matches)
  update()
  mql.addEventListener('change', update)
  onScopeDispose(() => mql.removeEventListener('change', update))
  return matches
}
```

- Read browser globals inside `onMounted` or behind a `typeof window` guard — at setup they run on the server too.
- An initial value that differs between server and client is a hydration mismatch waiting to happen: start with the server-safe value and update in `onMounted` (`ssr.md`).
- Every `addEventListener`, `setInterval`, `ResizeObserver`, `IntersectionObserver`, and `WebSocket` gets its removal registered in the same block where it was created. Distance between add and remove is how leaks are born.

## Composition and Boundaries

- Composables call composables; that is the point. Keep each one at a single responsibility so the combination is readable: `usePagination` + `useUsers`, not `useUsersWithPaginationAndFilters`.
- Do not compose in a loop or a condition — the effects and hooks registered depend on it, and the count must be stable for the component's lifetime.
- A composable that takes 6+ arguments and returns 10 keys is a component in disguise; split it or make it a store.

## Composable, Store, or Plain Function

| Signal | Reach for |
|---|---|
| No reactive state, no effects | A plain function — the `use` prefix on a pure helper is noise |
| Reactive state scoped to one component's lifetime | Composable |
| Shared state across sibling routes, devtools/HMR/SSR needed | Pinia store (`state.md`) |
| Shared read-only config down one subtree | `provide`/`inject` with an `InjectionKey` |
| Anything else | Start as a composable; promote to a store when a second unrelated feature reads it |

## VueUse

- Roughly 200 utilities covering the boring, easy-to-get-wrong cases (SSR guards, passive listeners, cleanup, ref-or-value inputs). Reimplementing `useIntersectionObserver` correctly costs more than the dependency for most teams.
- It is tree-shakeable — importing three functions does not ship the library.
- Extend rather than fork: most utilities accept a `MaybeRefOrGetter` target and an options object.
- Draw the line at trivial wrappers: a project-specific `useAuth` belongs in the project, and reaching for a dependency to avoid writing `ref(false)` is the other failure mode.

## Composable Review Checks

- Does it work when called with a plain value, a ref, and a getter?
- Does every effect and listener it creates stop when the caller unmounts?
- Does it run on a server without touching `window`?
- Does re-running the source abort the previous async work?
- Does the caller get refs (destructurable) and read-only state where mutation would be a bug?
