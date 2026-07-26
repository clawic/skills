# Routing — Vue Router 4 Without the Silent Failures

Vue Router reuses component instances aggressively and reports most failures by resolving a promise rather than throwing. Both behaviors produce bugs that look like nothing happened.

## The Instance-Reuse Rule

Navigating from `/users/1` to `/users/2` keeps the same component instance mounted: `setup()` does not re-run, `onMounted` does not fire, and data fetched in setup stays stale.

```ts
const route = useRoute()
watch(() => route.params.id, load, { immediate: true })   // the fix
```

- The blunt alternative — `<RouterView :key="$route.fullPath" />` — forces a remount on every navigation, including a query-string change, and throws away scroll position and transition state. Use it only where remounting is genuinely what you want.
- `onBeforeRouteUpdate(to, from)` is the in-component hook for the same event, and it can cancel.
- Destructuring `useRoute()` loses reactivity: `const { params } = useRoute()` freezes. Keep `route` and read `route.params.id` at the point of use.

## Route Definitions

```ts
{ path: '/users/:id', component: () => import('./UserView.vue'), props: true },
{ path: '/files/:path(.*)*', component: FileView },        // repeatable wildcard segment
{ path: '/:pathMatch(.*)*', component: NotFound }          // 404 catch-all (v3's '*' is gone)
```

- `props: true` passes params as props, which makes the component testable without a router. `props: route => ({ id: Number(route.params.id) })` also fixes the fact that every param is a string.
- Lazy `component: () => import(...)` is per-route code splitting — the highest-value one-line change in most apps (`performance.md`).
- Named routes (`name: 'user'`) survive path refactors; `router.push({ name: 'user', params: { id } })` does not break when the URL scheme changes.
- Nested routes need a `<RouterView />` in the parent component; a child route that renders nothing usually means the parent forgot it.
- `redirect` and `alias` differ: redirect changes the URL, alias serves the same component under two URLs. Choose alias only when both URLs are meant to be canonical for the user.

## Guards

| Guard | Runs | Typical job |
|---|---|---|
| `router.beforeEach` | Every navigation, before resolving | Auth, feature flags |
| `beforeEnter` (per route) | Only that route | Route-specific permission |
| `onBeforeRouteUpdate` | Same route, changed params | Refetch (see above) |
| `onBeforeRouteLeave` | Leaving the component | Unsaved-changes confirmation |
| `router.beforeResolve` | After all component guards, before confirming | Data prefetch that must block navigation |
| `router.afterEach` | After confirmation, cannot cancel | Analytics, title updates |

```ts
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !auth.isLoggedIn) {
    return { name: 'login', query: { next: to.fullPath } }   // redirect
  }
  // return true or nothing to allow; return false to abort
})
```

- Return a value or call `next()` — never both. Doing both leaves the navigation pending forever, and the page simply stops responding to links.
- A guard that redirects to a route whose own guard redirects back is an infinite loop; the router aborts it, but the user sees a frozen link. Always exempt the target route from the condition.
- `to.meta` inherits from parent routes, so `meta: { requiresAuth: true }` on a layout route covers all children.
- Async guards block navigation while they await. Keep them under a network round trip, or navigate first and show a loading state.

## Navigation Failures Are Not Errors

`router.push()` resolves with a `NavigationFailure` object instead of rejecting when a guard aborted, the target was the current route, or a newer navigation superseded it.

```ts
const failure = await router.push(target)
if (isNavigationFailure(failure, NavigationFailureType.aborted)) { /* handle */ }
```

- "Nothing happens when I click the link" is usually a duplicated navigation (`NavigationFailureType.duplicated`) — pushing the route the user is already on.
- A rejected promise from `push` means a guard threw. Register `router.onError` so those surface instead of dying in an unhandled rejection.

## Scroll Behavior

```ts
scrollBehavior(to, from, savedPosition) {
  if (savedPosition) return savedPosition            // back/forward restores position
  if (to.hash) return { el: to.hash, behavior: 'smooth' }
  return { top: 0 }
}
```

- Without `scrollBehavior`, an SPA keeps the old scroll offset on navigation — the "new page starts halfway down" complaint.
- Restoring position before async content renders scrolls to a page that is still short; return a promise from `scrollBehavior` that resolves after the data settles.

## History Mode and the Server

- `createWebHistory()` gives clean URLs and requires the server to serve `index.html` for unmatched paths. Its absence is the classic "works when I click, 404 when I refresh".
- `createWebHashHistory()` needs no server config and is the fallback for static hosts you do not control; the hash is never sent to the server, which breaks server-side redirects and some analytics.
- Deploying under a sub-path requires the base in both the router (`createWebHistory('/app/')`) and the bundler — mismatch produces assets that 404 or routes that never match.

## Params, Query, and State

- Params and query values are always strings (or arrays of strings for repeatable params). Coerce at the boundary, in `props`.
- Query is for shareable, bookmarkable state — filters, page number, search. It survives refresh and can be pasted into Slack; component state cannot.
- `history.state` via `router.push({ ..., state })` carries non-serializable-ish payloads without a URL, but it does not survive a hard refresh — never make it the only source.
- Never stuff an object into a query string as JSON. Put the id in the URL and refetch.

## Focus and Announcements on Navigation

An SPA navigation changes the document without a page load, so the browser does none of what it does on a real one: focus stays on the link the user just activated, and a screen reader announces nothing at all.

```ts
router.afterEach((to) => {
  document.title = `${to.meta.title ?? 'App'} — Acme`   // announced by some readers, and it fixes the tab and history
  nextTick(() => {
    const h = document.querySelector('main h1')
    if (h) { h.setAttribute('tabindex', '-1'); (h as HTMLElement).focus() }
  })
})
```

- Move focus to the new view's heading (or to the `<main>` landmark) after the view renders — `afterEach` fires before the component mounts, so the move belongs inside `nextTick` or in the view's own `onMounted`.
- `tabindex="-1"` makes a heading programmatically focusable without adding it to the tab order. Suppress the focus ring on it (`:focus:not(:focus-visible)`) or every navigation flashes an outline.
- A polite live region (`<div aria-live="polite">` fed the new page title) covers readers that ignore `document.title` changes. One region, at the app root, updated on `afterEach`.
- Skip-link behavior breaks the same way: the link's target must be focusable, or the reader keeps reading the nav on every route.
- Scroll restoration and focus are separate problems. `scrollBehavior` moves the viewport; only focus moves the keyboard and the reader.

## Transitions and KeepAlive with the Router

```vue
<RouterView v-slot="{ Component, route }">
  <Transition name="fade" mode="out-in">
    <KeepAlive :max="10" :include="['ListView']">
      <component :is="Component" :key="route.meta.keepScroll ? undefined : route.fullPath" />
    </KeepAlive>
  </Transition>
</RouterView>
```

- `mode="out-in"` avoids both views occupying the layout at once.
- Cached views do not re-run `onMounted` on return — refresh in `onActivated`.
- `KeepAlive`'s `include` matches the component's `name`; a `<script setup>` component takes its name from the filename, so a rename silently disables caching. `defineOptions({ name: 'ListView' })` makes it explicit.

## Route Design Checks

- Is every piece of state that the user would expect to survive a refresh in the URL?
- Does each component work when mounted directly at its URL, not only when reached by clicking?
- Does the auth guard exempt the login route and the 404 route?
- Is there a catch-all route, and does it render something better than a blank page?
- Does focus move to the new view on navigation, so a keyboard or screen reader user knows the page changed?
