# Plugins and the App Instance — app.use, Global Config, Registration

Vue 3 has no global `Vue` object: everything that was global is now per app instance. That is what makes two apps on one page, and one app per SSR request, safe — and what breaks code ported from Vue 2 that registered things at module load.

## The Install Contract

```ts
// A plugin is any object with install(app, options) — or a plain function with that signature.
export const analytics = {
  install(app, options) {
    app.provide(AnalyticsKey, createClient(options))   // preferred: typed, injectable, mockable
    app.component('TrackedLink', TrackedLink)
    app.directive('track', vTrack)
    app.config.globalProperties.$track = track        // last resort, template-only convenience
  }
}
app.use(analytics, { key: import.meta.env.VITE_ANALYTICS_KEY })
```

- `app.use` is idempotent per app: installing the same plugin twice is a no-op, so a library can call it defensively without guarding.
- Install order matters only where one plugin reads another's provide during install. Router and store first, everything that resolves a route or a store during install after.
- `provide` beats `globalProperties` on every axis that matters: typed through an `InjectionKey`, replaceable in a test by providing a different value, and absent from components that do not ask for it. `globalProperties` reaches only templates, needs a `ComponentCustomProperties` augmentation to type, and can never be tree-shaken.

```ts
// Typing globalProperties, in the plugin's own .d.ts
declare module 'vue' {
  interface ComponentCustomProperties { $track: (event: string) => void }
}
```

- Without that augmentation `$track` is `any` in every template and `vue-tsc` never catches a typo.
- Globally registered components are in the bundle whether or not a screen uses them. Register the handful that appear on most screens; import the rest locally.

## App-Level Config Worth Knowing

| Setting | Does | Trap |
|---|---|---|
| `app.config.errorHandler` / `warnHandler` | The global error and warning net | Returning without logging swallows the error (see the error guide) |
| `app.config.globalProperties` | Adds `$x` to every template | Invisible to imports, untyped without augmentation |
| `app.config.performance` | Marks component init and render in the browser's performance panel | Dev builds only |
| `app.config.compilerOptions` | Options for the **runtime** compiler (`isCustomElement`, whitespace) | Silently ignored by the SFC build — the same options must go in the bundler plugin's `template.compilerOptions` |
| `app.config.idPrefix` (`vue >=3.5`) | Namespaces `useId()` output | Two apps on one page collide on ids without it |
| `app.provide(key, value)` | App-wide injection, no ancestor needed | Feature state does not belong here — a store gives devtools and HMR |

## Multiple Apps and Per-Request Apps

- `app.config`, registered components, directives, and provides are per instance and are **not** shared. A widget mounted twice installs its plugins twice.
- Module-scope state inside a plugin is shared by every app that installs it — the same leak class as a module-scope store in SSR. Create state inside `install`, never beside it.
- SSR: build the app *and* install its plugins per request. A plugin instantiated at module load carries one user's client into another user's render.

## Teardown

- `app.unmount()` destroys the component tree and runs `unmounted` hooks; it does **not** tear down plugins. A plugin that opens a socket, starts an interval, or registers a `window` listener leaks across mount/unmount cycles — which is how a test suite ends with 40 live intervals.
- `app.onUnmount(cb)` (`vue >=3.5`) is the registration point for plugin cleanup. Below 3.5, return or expose a `destroy()` and make the host call it.

## When Not to Write a Plugin

- A plain module export is simpler for anything that does not need `app`. A `formatDate` helper imported where used beats `$formatDate` on every instance: it tree-shakes, it types itself, and its call sites are greppable.
- Reach for a plugin only to register components or directives, to provide app-wide config, or to touch `app.config`. Everything else is a composable or an import.
- Vue 2 `mixin`-shaped plugins port badly: `app.mixin` still exists, but a mixin's property collisions are silent, and the composable it becomes has explicit inputs and outputs.

## Plugin Review Checks

- Does the plugin create its state inside `install`, so two apps do not share it?
- Is anything it exposes reachable through `provide` with an `InjectionKey`, rather than only through `globalProperties`?
- If it adds `globalProperties`, does it ship the `ComponentCustomProperties` augmentation?
- Does everything it starts (sockets, intervals, listeners) stop on `app.onUnmount` or an exposed `destroy()`?
- Are globally registered components ones that most screens actually render?
