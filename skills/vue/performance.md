# Performance — Find the Cost Before You Optimize

Order of operations: measure, identify which of the four costs you have, then apply the fix for that cost. Applying `v-memo` to a bundle problem, or code-splitting a re-render problem, wastes the afternoon.

## Which Cost Do You Have?

| Symptom | Cost | Where to look |
|---|---|---|
| Typing or dragging lags, page already loaded | Re-render frequency | DevTools Performance timeline, "Highlight updates" |
| One interaction takes 300ms+ once | Single expensive render or computed | Timeline flame chart of that interaction |
| Blank screen before anything appears | Bundle / network | Network waterfall, bundle visualizer |
| Memory climbs across navigation | Leak, not slowness | `debug.md` Memory Grows |
| Slow only in dev | Dev-mode overhead (source maps, HMR, warnings) | Re-measure a production build before touching code |
| Anything else ("it feels slow") | Unclassified — you do not have a cost yet | Record one Performance trace of the exact interaction the user complains about; if nothing in it takes longer than a frame, the problem is perceived latency (no feedback on click, layout shift), not render cost |

Measure production builds. Dev mode adds warning checks and component instrumentation, so dev timings systematically overstate render cost and understate bundle cost.

## Re-Render Frequency

- Vue re-renders a component when a reactive value read during its render changes. Reading store state that changes often in a leaf component subscribes that leaf to every change — read narrowly (`storeToRefs` one field, or a `computed` of the slice).
- Compile-time optimizations already handle static content: the SFC compiler hoists static nodes and patch-flags dynamic bindings, so a mostly-static template is cheap without help. Do not hand-optimize what the compiler removed.
- Inline arrow handlers (`@click="() => select(item)"`) create a new function each render. This matters for a child component receiving it as a prop (new identity = new prop = child re-renders), and is harmless on a native element.
- Object and array props built inline (`:config="{ dense: true }"`) are new references every render for the same reason — hoist them to a `computed` or a module constant.
- `v-once` renders a subtree once and never updates it. `v-memo="[a, b]"` re-renders a subtree only when one of the listed values changes — and silently shows stale content if the list is incomplete, which is why it is a last resort, not a default.

## Large Lists

Escalate in this order:

1. **Stable keys.** Index keys force Vue to patch content across reordered instances instead of moving them (SKILL.md rule 7). This is correctness first, speed second.
2. **Shrink the row.** A row component with 3 watchers and a computed costs that per row. Move formatting into the parent's precomputed data.
3. **`shallowRef` the collection.** For thousands of rows replaced wholesale from an API, deep reactivity buys nothing.
4. **`v-memo` per row**: `<tr v-memo="[item.id, item.updatedAt, item.id === selectedId]">` — skips the row's patch entirely when nothing in the list changed. The selection flag is the one people forget, and its absence is why the highlight stops moving.
5. **Virtualize** above `virtualize_threshold` rows (default 200; see SKILL.md Configuration). Below it, virtualization adds scroll and accessibility complexity that costs more than it saves.
6. **Paginate or filter server-side.** If the user cannot meaningfully consume 10,000 rows, rendering them was the wrong requirement.

## Expensive Computation

- `computed` caches; a method called from the template does not — `{{ format(row) }}` runs on every render of that component.
- A computed depending on a frequently-changing ref recomputes at that frequency. Split it: derive a stable intermediate first, then the expensive step from the intermediate.
- Debounce the input, not the computation: `refDebounced`-style patterns keep the reactive graph intact, while a debounced watcher writing a second ref duplicates the state.
- Move genuinely heavy work (parsing, crypto, image processing) to a Web Worker and write the result into a ref. A 200ms synchronous computation blocks the frame no matter which Vue API wraps it.
- `markRaw` on large read-only datasets (lookup tables, GeoJSON) avoids proxying tens of thousands of nodes for data nobody mutates.

## Bundle Size

- Route-level splitting first: `component: () => import('./views/Reports.vue')`. It is one line per route and moves the most bytes.
- `defineAsyncComponent` for heavy, conditionally-rendered widgets (rich editors, charts, maps, date pickers) — the code loads when the widget renders.
- Prefetch on intent (hover, viewport entry) rather than on load; a `<link rel="prefetch">` or a plain call to the import function warms the chunk without blocking.
- Import members, not namespaces: `import { debounce } from 'lodash-es'` tree-shakes, `import _ from 'lodash'` does not.
- The runtime-only build (Vite's default for SFCs) excludes the template compiler. The compiler is only needed for templates provided as strings at runtime — and shipping it also forces `unsafe-eval` in your CSP (`security.md`).
- Inspect before cutting: a bundle visualizer plugin tells you whether the problem is your code or a 400 KB date library, and the answer changes the fix entirely.

## Startup and Perceived Speed

- Hydration cost scales with node count, not payload size: a page with 20,000 DOM nodes hydrates slowly even if the HTML gzips small (`ssr.md`).
- Defer below-the-fold work with `IntersectionObserver` + `defineAsyncComponent`; the user judges the first screen.
- `<Suspense>` centralizes loading states but delays the whole boundary until the slowest child resolves — narrower boundaries render sooner.
- Avoid a global "loading everything" gate: rendering the shell with skeletons beats a spinner on a blank page at identical total time.

## Profiling Checklist

- Vue DevTools "Highlight updates" — anything flashing that should not is your re-render list.
- DevTools Performance timeline — component render durations, ordered.
- `app.config.performance = true` marks component init/render in the browser's performance panel (dev only).
- Chrome heap snapshots, three navigations apart, for retained detached nodes (`debug.md`).
- Re-measure after each change, one change at a time. Two optimizations applied together tell you nothing about which one worked, and one of them is usually a regression.
