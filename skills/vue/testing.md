# Testing — Test Utils, Async Timing, and What to Mock

The two failure modes are asserting before Vue has updated, and testing the implementation instead of the contract. Both produce suites that pass while the app is broken.

## Async Timing

Every DOM assertion needs the update to have flushed. Vue batches; nothing is synchronous.

```ts
await wrapper.find('button').trigger('click')   // trigger returns a promise — await it
await nextTick()                                 // one flush: state → DOM
await flushPromises()                            // pending microtasks too: fetch mocks, awaits
expect(wrapper.text()).toContain('Saved')
```

- `trigger`, `setValue`, `setProps`, and `setData` all return promises that resolve after the next tick. A missing `await` is the single most common flaky-test cause in Vue suites.
- One `nextTick` flushes one render cycle. A chain (click → watcher → fetch → state → render) needs `flushPromises()`, which drains the microtask queue.
- `vi.useFakeTimers()` plus `await vi.advanceTimersByTimeAsync(300)` for debounced code; the sync `advanceTimersByTime` does not let awaited promises resolve.
- Never `await new Promise(r => setTimeout(r, 50))` to "let it settle" — it hides the race and makes the suite slow and still flaky.

## Mounting

```ts
const wrapper = mount(UserCard, {
  props: { user },
  global: {
    plugins: [createTestingPinia({ initialState: { user: { current: user } } })],
    stubs: { RouterLink: true, Teleport: true },
    mocks: { $t: (k: string) => k }
  },
  attachTo: document.body      // required for focus, measurement, and real event bubbling
})
```

- `mount` renders children; `shallowMount` stubs them. Default to `mount` — stubbed children hide integration bugs, and `shallowMount` earns its place only when a child is heavy (a chart, a map) or has a hard dependency.
- `attachTo: document.body` is required for anything touching focus, `document.activeElement`, `getBoundingClientRect`, or teleported content — and then `wrapper.unmount()` in cleanup, or the nodes accumulate across tests.
- `createTestingPinia()` stubs actions by default so you can assert they were called; pass `{ stubActions: false }` when the test needs the real action to run.
- Stub `RouterLink`/`RouterView` for a component test; install a real router only when the test is about routing.

## Query and Assert on the Contract

```ts
wrapper.get('[data-test="submit"]')        // get throws if absent — better failure message than find
wrapper.find('.btn').exists()
wrapper.emitted('select')                  // [[arg], [arg]] — array of call arg arrays
expect(wrapper.emitted('select')?.[0]).toEqual([{ id: '1' }])
```

- Query by role or `data-test` attribute, not by CSS class — a class is a styling decision that will change without a behavior change.
- Assert on rendered output and emitted events, never on `wrapper.vm.internalFlag`. A test that reads internals fails on every refactor and passes through every real regression.
- `wrapper.html()` in a failing assertion's message beats guessing; print it once, then delete the debug line.

## Testing Composables

```ts
// Pure composable: call it directly
const { count, increment } = useCounter(3)

// Uses lifecycle hooks or inject: needs a host component
function withSetup<T>(fn: () => T): [T, App] {
  let result!: T
  const app = createApp({ setup() { result = fn(); return () => {} } })
  app.mount(document.createElement('div'))
  return [result, app]          // app.unmount() triggers onUnmounted cleanup
}
```

- A composable with no lifecycle hooks and no `inject` is a plain function — test it as one, with no mounting overhead.
- Test the cleanup path explicitly: mount, assert the listener was added, unmount, assert it was removed. Leaks are invisible otherwise.
- Test the async race: change the source twice in a row and assert the first response cannot overwrite the second (`composables.md`).

## jsdom Gaps

| Missing | Symptom | Handling |
|---|---|---|
| `IntersectionObserver`, `ResizeObserver` | `not defined` on mount | Stub globally in the setup file |
| `matchMedia` | `not a function` | Stub with a fixed `matches` value |
| Layout: every rect is 0×0 | Virtual scrollers render nothing; measurement code takes the wrong branch | Mock `getBoundingClientRect`, or move the test to a browser runner |
| `scrollIntoView`, `HTMLDialogElement.showModal` | `not a function` | Stub, or use a real browser environment |
| CSS: no cascade or layout | Visibility and transition assertions are meaningless | Assert classes and state, not computed styles |

When the test depends on real layout or real browser APIs, that is the signal it belongs in an end-to-end runner (`playwright`), not in a component test with five more stubs.

## What to Test at Which Level

| Level | Covers | Keep out |
|---|---|---|
| Unit (plain functions, composables) | Logic, edge cases, error paths | DOM |
| Component (Test Utils) | Props in → rendered output and events out; user interaction | Router config, real network, layout |
| End-to-end | Navigation, auth flows, real backend contracts | Every prop permutation |

Rule of thumb for the mix: exhaustive cases at the unit level, one representative interaction per component, and end-to-end only on the paths whose breakage would be an incident.

## Snapshots

- Whole-component snapshots break on every markup edit and get accepted without reading. Their value approaches zero, and negative once the team learns to press `u`.
- Inline snapshots of a small, meaningful projection (the rendered rows' text, the emitted payload) stay readable and diff usefully.

## Jest Instead of Vitest

Test Utils itself is runner-agnostic — `mount`, `trigger`, `emitted`, and `createTestingPinia` behave identically. What changes is the mocking API and the build config that Vitest inherits from Vite for free.

```js
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',                    // Jest defaults to node: without this, `document` is undefined
  transform: { '^.+\\.vue$': '@vue/vue3-jest', '^.+\\.[jt]sx?$': 'babel-jest' },
  transformIgnorePatterns: ['/node_modules/(?!(vue-router|pinia|@vueuse)/)'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1', '\\.(css|scss)$': 'identity-obj-proxy' }
}
```

- Mocking maps one to one: `vi.fn`, `vi.mock`, `vi.spyOn`, `vi.useFakeTimers` → `jest.*`. `vi.advanceTimersByTimeAsync` needs `jest >=29.5` for `jest.advanceTimersByTimeAsync`; below that there is no async-aware advance and a debounce test has to await the promise chain by hand.
- `SyntaxError: Unexpected token 'export'` from a dependency is `transformIgnorePatterns`: Jest skips all of `node_modules` by default, so every ESM-only package has to be listed back in.
- Path aliases and CSS/asset imports need `moduleNameMapper`. Vitest reads `resolve.alias` from the Vite config and handles style imports itself, which is most of why its config is three lines.
- `import.meta.env` does not exist under Jest's CJS transform — read env through one wrapper module you can mock, or every SFC that touches it throws at import time.
- A project keeping Jest only out of inertia is a one-afternoon migration: the specs move unchanged, and the config above collapses to `environment: 'jsdom'`.

## Testing Review Checks

- Is every interaction awaited, with no arbitrary `setTimeout` in the suite?
- Do assertions read rendered output and emitted events rather than `vm` internals?
- Are queries by role or `data-test`, not by styling class?
- Does the suite cover a cleanup case (unmount removes listeners) and an async race?
- Is `mount` the default, with stubbing justified per case?
