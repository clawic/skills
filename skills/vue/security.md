# Security — XSS, Template Injection, and Client-Side Trust

Vue escapes mustache interpolation, so the framework's default is safe. Every hole below is a place where you opted out of that default, usually to render something a user supplied.

## The Injection Surfaces

| Surface | Risk | Rule |
|---|---|---|
| `v-html` | Full XSS: scripts, event handlers, `<img onerror>` | Never with user-influenced content. If rich text is a product requirement, sanitize server-side with an allowlist and store the sanitized version |
| `:href` / `:src` | `javascript:` and `data:text/html` URLs execute on click | Validate the scheme (`https:`, `mailto:`, relative) before binding; reject everything else |
| `<component :is>` | Renders any component reachable from the map or registry | Map user input through a fixed allowlist object; never `:is="userString"` |
| Runtime `template:` strings | Template compilation is code execution — full JS, not just markup | Never build a template from user data; prefer the runtime-only build so the compiler is not even shipped |
| `:style` with a user string | CSS injection: exfiltration via `background-image: url(...)`, overlay tricks | Bind an object of known properties, not a string |
| Dynamic `v-bind="userObject"` | Spreads arbitrary attributes, including event handlers in some renderers | Pick fields explicitly |
| SSR state serialization | A payload containing `</script>` breaks out into executable HTML | Escape `<` as `\u003c` before injecting (`ssr.md`) |

- Sanitizing on the client is a UX nicety, not a control: an attacker calls your API directly. Sanitize where the data is stored.
- A sanitizer allowlist must cover attributes too — `<a href="javascript:...">` and `<img onerror>` pass any tag-only filter.

## Content Security Policy

- A strict CSP without `unsafe-eval` requires the runtime-only build (templates precompiled at build time). The full build compiles templates with `new Function`, which the CSP blocks — the symptom is a blank page and a CSP violation, not a Vue warning.
- `v-bind` to inline styles and scoped-CSS `v-bind()` generate inline style attributes; a policy without `'unsafe-inline'` for styles needs nonces or hashes configured in the bundler.
- Report-only mode first, then enforce: a CSP added at the end of a project always breaks a third-party widget nobody remembers adding.

## Client-Side Trust Boundaries

- Route guards are UX, not authorization. `meta: { requiresAdmin: true }` hides a link; the API must reject the request independently. Anyone can call the endpoint directly.
- Everything in the bundle is public: API keys, feature-flag logic, hidden routes, and commented-out code all ship. `VITE_`-prefixed env vars are inlined into the client build by design — a secret placed there is published.
- `v-if="user.isAdmin"` removes elements from the DOM, which stops casual inspection and nothing else.
- Validation rules are duplicated deliberately: the client version is feedback, the server version is the control (`forms.md`).

## Token and Session Handling

- `localStorage` is readable by any script on the origin, so one XSS equals full token theft with no expiry. `httpOnly` cookies with `SameSite=Lax` (or `Strict`) keep tokens out of JS entirely; the tradeoff is CSRF protection you must then implement.
- Never persist tokens or PII in a Pinia store with a persistence plugin — that writes them to `localStorage` by default (`state.md`).
- On logout, clear store state and abort in-flight requests; a pending fetch resolving after logout repopulates the store with the previous user's data.

## Dependencies and Supply Chain

- Declare and pin: a Vue app's attack surface is mostly its dependency tree. A UI library update that adds an analytics beacon is a real occurrence, not a hypothetical.
- Third-party widgets mounted into the app's DOM run with full access to the page — a chat widget can read any field. Isolate untrusted embeds in an iframe with `sandbox`.
- `dangerouslySetInnerHTML`-style helpers imported from a component library carry the same `v-html` risk under a different name; audit for them by behavior, not by name.

## Security Review Checks

- Grep the project for `v-html` and justify every occurrence, or replace it.
- Does any `:href`, `:src`, or `:is` receive a value that originated outside the codebase?
- Is the production build runtime-only, with no runtime template compilation?
- Are tokens outside `localStorage`, and is nothing sensitive in the SSR state payload?
- Does every guarded route have a server-side check behind it?
