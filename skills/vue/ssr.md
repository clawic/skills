# SSR — Hydration, Request Isolation, and Universal Code

Scope: Vue's own SSR (`createSSRApp` + `renderToString`, or a Vite SSR setup). Nuxt's layer — auto-imports, `useFetch`/`useAsyncData`, `<ClientOnly>`, server routes, nitro — is `nuxt`; the mechanics below still explain what Nuxt is doing.

Two rules generate almost every SSR bug: the server renders once with no DOM, and the module scope is shared by every request.

## Request Isolation

```ts
// entry-server.ts — a factory per request, never a module-scope app
export function createApp() {
  const app = createSSRApp(App)
  const router = createRouter({ history: createMemoryHistory(), routes })
  const pinia = createPinia()
  app.use(router).use(pinia)
  return { app, router, pinia }
}
```

- A module-scope `const state = reactive({})`, a singleton store, or a module-level `createRouter()` is shared across concurrent requests: one user's data renders into another user's HTML. This is a data-leak class of bug, not a performance detail.
- `createMemoryHistory()` on the server, `createWebHistory()` on the client — a browser history on the server throws on `window`.
- The same rule reaches composables: a `use*` that keeps refs at module scope is a global (`composables.md`).

## State Transfer

```ts
// server: after render
const payload = JSON.stringify(pinia.state.value).replace(/</g, '\\u003c')
html = html.replace('<!--state-->', `<script>window.__PINIA__=${payload}</script>`)
// client: before mount
if (window.__PINIA__) pinia.state.value = window.__PINIA__
```

- Escape `<` in the serialized payload (as `\u003c`) or a string containing `</script>` closes the tag and executes whatever follows — a stored-XSS vector (`security.md`). `U+2028`/`U+2029` also need escaping for older parsers.
- Hydrate the store BEFORE mounting the app; after mount, the first render already used empty state and you get a mismatch.
- Never serialize secrets: everything in the payload is visible in view-source. Filter server-only fields before transfer.

## Hydration Mismatches

The client's first render must produce the same tree as the server's HTML. When it does not, Vue warns and — on `vue >=3.5` — reports the offending node with a clearer diff.

| Cause | Example | Fix |
|---|---|---|
| Non-deterministic values | `Date.now()`, `Math.random()`, `crypto.randomUUID()` in a render | Compute on the server and pass down; `useId()` (`vue >=3.5`) for ids |
| Browser-only reads at setup | `window.innerWidth`, `localStorage`, `navigator.language` | Server-safe initial value, update in `onMounted` |
| Locale / timezone drift | Server in UTC formats a date differently than the browser | Send an ISO string, format after mount, or pin the timezone explicitly |
| Invalid HTML nesting | `<div>` inside `<p>`, `<tr>` outside `<tbody>`, block inside `<button>` | The browser silently repairs the DOM, so the tree differs before your code runs — fix the markup |
| Extensions and injected scripts | Password managers and translators inject nodes | Not your bug; verify in a clean profile before investigating |
| Conditional client-only branches | `v-if="isClient"` where `isClient` is true at first client render | Start `false`, flip in `onMounted` |

- A mismatch is not cosmetic: Vue discards the server DOM for the affected subtree and re-renders it on the client, which costs the exact performance SSR was bought for.
- The deliberate escape hatch is a client-only wrapper: render nothing on the server, mount after `onMounted` sets a flag.

## Lifecycle on the Server

- Only `setup` runs. `onMounted`, `onUpdated`, `onUnmounted` and everything DOM-facing never fire server-side — which makes `onMounted` the standard home for browser work, and means cleanup registered only in `onUnmounted` never runs on the server (a server-side `setInterval` in setup leaks per request).
- Watchers do not run on the server (there is no update cycle), so state derivation that only happens in a watcher produces empty HTML. Derive with `computed`, which does evaluate during render.
- Teleports render into a separate bucket that must be injected into the HTML explicitly; a teleport whose target is outside the app root silently disappears from the server output.
- `async setup()` under `<Suspense>` works in SSR, but the whole boundary waits for the slowest child before any of it streams.

## Universal Code Patterns

```ts
const isClient = typeof window !== 'undefined'
if (import.meta.env.SSR) { /* server-only branch, tree-shaken from the client bundle */ }
```

- `import.meta.env.SSR` is compile-time and removes the branch from the client bundle; `typeof window` is runtime and keeps both. Use the first for excluding server code, the second inside shared utilities.
- Never import a Node-only module (`fs`, `path`, a database client) at the top level of a component or a shared composable — the bundler follows the import into the client bundle and the build fails or ships a polyfill.
- Cookies and headers are per-request state: pass them through a request context object, never a module-scope variable.

## Performance Notes

- Hydration cost tracks DOM node count, not payload bytes. A 20,000-node page hydrates slowly however small the gzip (`performance.md`).
- `renderToString` is synchronous work on the server thread — a slow component blocks other requests. Cache what is cacheable at the HTTP layer before optimizing components.
- SSG is SSR run at build time; every rule above applies except request isolation, which becomes build-time isolation (a module-scope cache that persists across pages produces one page's data on another).

## SSR Review Checks

- Is every app, router, and store created inside a per-request factory?
- Does any component read `window`, `document`, or `localStorage` during setup?
- Is transferred state escaped, and free of server-only fields?
- Does the first client render match the server's, including ids, dates, and conditional branches?
- Is the markup valid HTML, so the browser does not repair it before hydration?
