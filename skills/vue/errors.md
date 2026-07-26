# Errors — Boundaries, the App Handler, and Failure States

Vue ships no default error UI. An uncaught throw in setup or render unmounts that subtree and leaves a blank area plus one console line — no warning, no fallback. Every layer below is opt-in, and a production app needs all three: a global handler, boundaries around independently-failing regions, and a designed failure state per async widget.

## What Catches What

| Layer | Catches | Misses |
|---|---|---|
| `app.config.errorHandler` | Errors from setup, render, watchers, computed getters, lifecycle hooks, custom directive hooks, and event handlers — anywhere in the app | Code Vue never called: `setTimeout` callbacks, unhandled promise rejections, throws before `createApp`, navigation guard throws |
| `onErrorCaptured(err, instance, info)` | The same set, but only from this component's descendants — the boundary | Its own setup and render errors (a boundary cannot catch itself); async work no descendant's render owns |
| `router.onError` | Throws inside navigation guards and failures loading a lazy route chunk | Everything after the view mounts |
| `window.onunhandledrejection` | Every promise rejection nobody handled, including the ones the three above never see | Nothing — it is the last net, and belongs next to the app handler |

```ts
app.config.errorHandler = (err, instance, info) => {
  // info names the phase: "setup function", "render function", "watcher callback", "native event handler"
  report(err, { component: instance?.$options.__name, phase: info })
  if (import.meta.env.DEV) console.error(err)
}
app.config.warnHandler = (msg, instance, trace) => { /* dev only — stripped from production builds */ }
window.addEventListener('unhandledrejection', e => report(e.reason, { phase: 'unhandled rejection' }))
```

- An `errorHandler` that returns without logging **swallows the error**: it never reaches the console, and debugging becomes archaeology. Always re-log in dev.
- It fires per component instance. One broken computed in a 500-row list reports 500 times — dedupe by message plus component before shipping to a reporting service.
- `info` is the cheapest triage signal in the codebase: "setup function" means the component never produced a tree, anything else means it rendered at least once.

## The Boundary Component

```vue
<script setup>
const error = ref(null)
onErrorCaptured((err) => { error.value = err; return false })   // false stops the bubble here
</script>
<template>
  <slot v-if="!error" />
  <FailedPanel v-else :error="error" @retry="error = null" />
</template>
```

- Return `false` to stop propagation; returning nothing lets the error continue to the next boundary and then to `app.config.errorHandler`. Reporting wants the bubble, a widget that must not take the page down wants `false`.
- Clearing `error` re-renders the slot, and a component that throws deterministically throws again on the spot. Pair the retry with a `:key` bump or a refetch, never a bare reset.
- A boundary only wraps descendants. Put it around the risky subtree, not inside it.
- `<Suspense>` is not an error boundary: a rejected `async setup()` propagates past it, so a Suspense boundary needs a sibling `onErrorCaptured` or the fallback hangs forever.
- Boundaries are per region, not per app: one around the whole router view converts every bug into a full-page error screen, which is worse than a broken sidebar.

## The Page Went Blank

One console error, no Vue warning, nothing rendered. Ordered by probability:

1. Read the phase from `info` or the top stack frame. Setup-phase means no tree was ever produced; render-phase means it worked once and a later state broke it.
2. The innermost component in the trace is the culprit — a throw in setup unmounts everything below the nearest boundary, so the blank region is much larger than the bug.
3. `Cannot read properties of undefined` on first render is almost always data assumed present: a prop with no default, a store field before its fetch resolves, a `route.params` key this route does not carry.
4. Blank with **no** console error is not an error at all: an empty `v-for`, a false `v-if`, a dynamic import swallowed by an empty `.catch()`, or a CSP blocking the runtime compiler.
5. Blank only in production, with a chunk name in the error → chunk load failure after a deploy (next section).
6. Remove the boundary temporarily. A boundary whose fallback renders nothing is indistinguishable from a crash, and hides the console error that would have named the cause.

## Async Components and Chunk Failures

```ts
const Editor = defineAsyncComponent({
  loader: () => import('./Editor.vue'),
  errorComponent: LoadFailed,      // receives the error as a prop
  timeout: 10000,                  // without it, a stalled loader hangs forever with no error
  onError(err, retry, fail, attempts) {
    if (attempts <= 2 && isNetworkError(err)) return retry()
    fail()
  }
})
```

- The three settings cover three different failures: `errorComponent` a rejected loader, `timeout` a loader that never settles, `onError` the retry decision. An `errorComponent` alone still hangs on a dead network.
- `Failed to fetch dynamically imported module` in production means the chunk hash no longer exists on the CDN — a tab left open across a deploy. Handle it once globally: reload the page on that message, guarded by a `sessionStorage` flag so a genuine 404 cannot loop.
- Errors thrown *inside* an async component after it loads are ordinary render errors. They belong to a boundary, not to the loader options.
- `retry()` re-invokes the loader, so an import that fails for a syntax error retries pointlessly. Gate it on the error being network-shaped, as above.

## Designing the Failure State

- Three states, always: loading, error, empty. A component with only loading and success renders the same blank box for "request failed" and "no results", and the user cannot tell which happened.
- The error state carries the user's next action (retry, go back, who to contact) — never the stack trace. Log the trace, show the action.
- Distinguish transient from terminal: a 500 or a timeout gets a retry button, a 403 gets a route change, a 404 gets an empty state. A retry button on a 403 is the most common wrong error UI.
- Preserve user input on failure. A submit that throws must leave the form filled and re-enable the button, which is why the reset belongs in `finally`.
- Optimistic updates need a rollback path written at the same time as the update, or a failed request leaves the UI showing a change the server rejected.

## Error Handling Review Checks

- Is `app.config.errorHandler` installed, reporting, and still logging in dev?
- Is there an `unhandledrejection` listener, given the app handler never sees rejections?
- Does a boundary wrap each independently-failing region — widget, route view, modal — rather than the whole app?
- Does every `defineAsyncComponent` set `errorComponent` and `timeout`, and is post-deploy chunk failure handled once, globally, with a loop guard?
- Does every async region render distinguishable loading, error, and empty states?
- Does every failure path leave the user able to act: retry, navigate away, or keep the data they typed?
