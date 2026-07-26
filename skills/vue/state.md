# State — Pinia, provide/inject, and Where State Should Live

Default hierarchy: component `ref` → lifted to the common parent → `provide`/`inject` for a subtree → Pinia store for anything crossing routes or features. Each step outward buys reach and costs traceability; skipping straight to a store makes every value global and every bug app-wide.

## Pinia Store Shapes

```ts
// Setup store — full Composition API inside
export const useCart = defineStore('cart', () => {
  const items = ref<Line[]>([])
  const total = computed(() => items.value.reduce((s, l) => s + l.qty * l.price, 0))
  function add(line: Line) { items.value.push(line) }
  function $reset() { items.value = [] }          // setup stores have no built-in $reset
  return { items, total, add, $reset }
})

// Options store — state/getters/actions, gets $reset() for free
export const useUi = defineStore('ui', {
  state: () => ({ sidebarOpen: false }),
  getters: { isCompact: (s) => !s.sidebarOpen },
  actions: { toggle() { this.sidebarOpen = !this.sidebarOpen } }
})
```

- Setup stores compose composables and are easier to type; options stores get `$reset()` and read more like Vuex for a migrating team. Pick one shape per project.
- `state` must be a function, always — an object literal is shared across every app instance and every SSR request.
- Everything returned from a setup store becomes store state or an action; refs you meant to keep private must not be returned.

## The Five Pinia Traps

| Trap | Symptom | Fix |
|---|---|---|
| Calling `useStore()` at module scope | `"getActivePinia()" was called but there was no active Pinia` (Pinia <2.1: `getActivePinia was called with no active Pinia`) | Call inside setup, a router guard body, or after `app.use(pinia)` |
| Destructuring state | Values freeze at read time | `const { items } = storeToRefs(store)` for state and getters; plain destructure for actions |
| One store instance in SSR | User A's data appears for user B | `createPinia()` per request (`ssr.md`) |
| `$reset()` on a setup store | Runtime error — the method does not exist | Write `$reset` yourself and return it, as above |
| Two copies of `pinia` or `vue` in the tree | Components see different state; nothing is shared | Deduplicate the dependency before debugging your code |

## Store APIs Worth Knowing

- `$patch(partial)` or `$patch(state => { ... })` batches multiple mutations into one devtools entry and one subscriber notification — the function form is required for array splices.
- `$subscribe((mutation, state) => ...)` fires on state change; `$onAction(ctx => ...)` wraps actions with `after` and `onError` hooks. Persistence and audit logging belong here, not sprinkled through actions.
- `$dispose()` stops a store's effect scope. Needed for dynamically created stores (per-entity stores keyed by id), otherwise they accumulate.
- Store composition: call another store inside an action or a getter, never at store-definition top level in a circular pair — two stores that call each other at definition time deadlock on initialization order.
- Pinia plugins receive `{ store, app, pinia, options }` and can add properties to every store; that is the correct home for persistence, not a `watch` per store.

## Vuex 4, for Codebases That Still Have It

Vuex 4 is the Vue 3-compatible release and is in maintenance mode — no new features, Pinia is the recommended store. Keep it while a migration is in flight; never start a new store in it.

```ts
const store = createStore({
  modules: { cart: { namespaced: true, state: () => ({ lines: [] }), getters, mutations, actions } }
})
app.use(store)

// in setup
const store = useStore()                              // there is no `this.$store` in <script setup>
const lines = computed(() => store.state.cart.lines)  // computed, never a destructure
store.dispatch('cart/add', line)                      // namespaced path: a string nothing type-checks
```

- `const { lines } = store.state` freezes the value exactly as destructuring a Pinia store does (SKILL.md Reactivity Loss). Every read goes through a `computed`; there is no `storeToRefs` equivalent.
- `mapState` / `mapGetters` produce Options API objects and do nothing in `<script setup>` — wrap each field in its own `computed`.
- Mutations are synchronous by contract: an `await` inside one makes devtools time-travel report a state the app never had. Async work goes in an action that commits.
- A typo in a namespaced path is not an error — Vuex logs `unknown action type: cart/add` in dev and nothing in production. Export the strings as constants if the modules outlive the migration.
- `state` must be a function in every module, and `createStore()` runs per request in SSR — the same shared-instance leak as Pinia (`ssr.md`).
- Porting to Pinia: one namespaced module becomes one store, mutations collapse into actions, getters stay getters. Sequence it after the Vue 3 upgrade lands, never during it.

## Persistence

```ts
// Explicit, no plugin: hydrate once, save on change
const saved = localStorage.getItem('cart')
if (saved) items.value = JSON.parse(saved)
watch(items, v => localStorage.setItem('cart', JSON.stringify(v)), { deep: true })
```

- Read on the client only — `localStorage` at store creation runs on the server in SSR and throws.
- Restoring a persisted shape after a schema change is where this pattern breaks: version the payload (`{ v: 2, data }`) and drop what you cannot migrate, rather than merging a stale shape into a new store.
- Never persist tokens or PII in `localStorage`: it is readable by any script on the origin (`security.md`).
- Persisting a deep-watched large store serializes it on every keystroke. Debounce, or persist a narrow slice.

## provide / inject

```ts
// keys.ts
export const ThemeKey = Symbol('theme') as InjectionKey<Ref<Theme>>
// ancestor
provide(ThemeKey, theme)                 // provide the ref, not theme.value
// descendant
const theme = inject(ThemeKey)           // Ref<Theme> | undefined
const theme2 = inject(ThemeKey, defaultTheme)          // with default
const theme3 = inject(ThemeKey, () => makeTheme(), true) // factory default
```

- Provide the ref; providing `.value` hands down a snapshot no consumer ever sees change.
- `InjectionKey<T>` gives the injected value its type and eliminates string-key collisions across libraries.
- `inject()` must run synchronously in setup. In a callback or after `await` it warns "can only be used inside setup()".
- Give consumers write access through a provided function, not by providing a mutable ref and hoping: `provide(ThemeKey, { theme: readonly(theme), setTheme })` keeps the mutation path traceable.
- `app.provide()` covers the whole app — the right tool for a config object every component may read, and the wrong tool for feature state (a store gives you devtools and HMR).

## Choosing the Container

| The state is | Put it in | Because |
|---|---|---|
| One component's UI detail (open, hovered, draft text) | `ref` in that component | Nothing else can break it |
| Shared by two siblings | The common parent, passed down | Cheapest thing that works |
| Read by a whole subtree, written by its root (theme, form context, table config) | `provide`/`inject` | Reach without globality |
| Server data keyed by URL | A query cache or a store keyed by id | Deduplication and invalidation matter more than storage |
| Session-wide (user, permissions, cart, feature flags) | Pinia store | Devtools, HMR, SSR safety, plugin surface |
| Derived from other state | Nowhere — `computed` | Stored derived state is a second source of truth that drifts |
| Anything else | Start local, promote when a second feature reads it | Promotion is a small refactor; demotion never happens |

## Store Design Checks

- Does any store field duplicate something derivable? Delete it and add a getter.
- Does an action do UI work (toast, navigate)? Return a result and let the caller decide; a store that navigates cannot be reused on another screen.
- Is server data cached in a store with no invalidation story? Decide staleness explicitly, or use a query library and keep the store for client state.
- Does the store hold a component instance, a DOM node, or a class instance? `markRaw` it or move it out (SKILL.md rule 6).
