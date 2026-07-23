# Browser — DOM Events, Fetch, Storage, Loading

## Events

- Delegate: one listener on a stable container, `e.target.closest("[data-action]")` — survives re-renders and replaces N listeners with one.
- Removal requires the SAME function reference — an inline arrow can never be removed. Modern group teardown: pass `{signal}` from one `AbortController` to many listeners and abort once (also the listener-leak fix — `memory-leaks.md`).
- `{passive: true}` on scroll/touch listeners lets the browser scroll without waiting for your handler; `preventDefault` inside a passive listener is ignored (with a console warning), not an error you'll catch.
- `{once: true}` self-removes after the first call — init and first-interaction handlers.
- `input` fires per keystroke; `change` fires on commit (blur, enter, selection). Form values are ALWAYS strings — `el.valueAsNumber` for number/date inputs, or parse explicitly (`coercion.md`).
- `preventDefault` stops the default action; `stopPropagation` stops bubbling — orthogonal. `stopPropagation` also hides the event from delegation and analytics listeners upstream: prefer handling precisely over silencing broadly.

## fetch

- fetch rejects ONLY on network failure, CORS, or abort — 404 and 500 RESOLVE. Check `res.ok` before `.json()`; the unchecked-ok path is the single most common fetch bug.
- A body is single-use: a second `.json()`/`.text()` throws "body already read" — `res.clone()` before reading twice.
- `.json()` on an empty body (204, some error responses) throws — gate on status or read `.text()` first (`json.md` Safety).
- CORS failures are opaque to JS by design (no status, no detail) — the fix is response headers on the server, never client code. `mode: "no-cors"` is not a workaround: it yields an unreadable opaque response.
- No default timeout: `AbortSignal.timeout` (SKILL.md Traps; combinators in `async.md`).
- Cross-origin cookies need BOTH `credentials: "include"` and the server's `Access-Control-Allow-Credentials` — one without the other silently sends nothing.
- Exit-time reporting: `navigator.sendBeacon` or `fetch(url, {keepalive: true})` — an ordinary fetch is killed with the page.

## Storage

| Store | Size | Blocking | Sent to server | For |
|---|---|---|---|---|
| localStorage | ~5 MB, strings | sync, main thread | no | small persistent prefs |
| sessionStorage | ~5 MB, strings | sync | no | per-tab state |
| cookies | ~4 KB | sync | on EVERY request | server-read state only |
| IndexedDB | large (origin quota) | async | no | structured data, blobs, offline |

- Web storage stores strings: objects arrive as `"[object Object]"` unless you JSON-encode both ways (`json.md`).
- Never store tokens or secrets in localStorage — any XSS reads it all; httpOnly cookies are invisible to JS by design, which is the point.
- Writes can throw (quota, private-browsing modes) — wrap in try/catch and degrade; storage is a cache, not a guarantee.
- The `storage` event fires in OTHER same-origin tabs only — free cross-tab sync; within the same tab use `BroadcastChannel` or your own events.

## Script Loading & Page Lifecycle

- `type="module"`: deferred by default, strict mode, own scope — the modern default. Classic scripts: `defer` preserves order and runs after parse; `async` runs whenever loaded, order unpredictable — only for independents (analytics).
- `DOMContentLoaded` = DOM parsed; `load` = all resources fetched. Deferred/module scripts run before `DOMContentLoaded`. A `getElementById` returning null usually means the script ran before its element was parsed (or the id is simply wrong — check that first).
- "User left" hooks: `visibilitychange` → hidden is the reliable one; `unload` doesn't fire on mobile and disables bfcache; `beforeunload` is only for the unsaved-changes prompt.
- Background tabs throttle timers (SKILL.md Timers) — clocks and polls recompute from `Date.now()` on `visibilitychange`, never accumulate intervals.

## DOM Correctness

- User text into markup: `textContent`, or sanitize at ONE render boundary (SKILL.md Traps — innerHTML XSS).
- `getElementsByClassName`/`children` are LIVE collections — mutating while iterating skips elements. `querySelectorAll` is static; iterate that, or copy first (`[...collection]`).
- A NodeList has `forEach` but not `map`/`filter` — spread into an array before chaining.
- Build URLs with `new URL` and `URLSearchParams` — encoding of `&`, `+`, spaces, and Unicode handled; string concatenation breaks on all four.
- Batching reads/writes and animation scheduling: `performance.md` Browser Rendering.
