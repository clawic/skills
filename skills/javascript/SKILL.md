---
name: javascript
slug: javascript
version: 1.0.6
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
description: >-
  Writes, debugs, and reviews JavaScript: async and the event loop, coercion, closures, dates, Unicode, regex, and modern ES2023+ APIs.
  Use when JS throws TypeError or "undefined is not a function", a promise never settles or a rejection goes unhandled, NaN or
  [object Object] appears, dates shift by a day, sorting or equality misbehaves, memory grows, a regex hangs, JSON loses precision,
  a Node process won't exit or ignores signals, or fetch doesn't reject on a 404;
  also when choosing data structures, handling errors, profiling slow code, or checking whether a feature is safe for a target Node or
  browser version. Covers Node and browser runtime edges. Not for TypeScript type-system design or framework internals.
homepage: https://clawic.com/skills/javascript
metadata:
  clawdbot:
    emoji: 🟨
    displayName: JavaScript
    configPaths:
    - ~/Clawic/data/javascript/
---

User preferences and memory live in `~/Clawic/data/javascript/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/javascript/` or `~/clawic/javascript/`), move it to `~/Clawic/data/javascript/`.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/javascript/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| runtime_target | node \| browser \| both | node | Selects which platform file applies by default (`node.md` vs `browser.md`) and which floor gates advice |
| node_floor | number (Node major) | from package.json `engines`, else 22 | Gates every recommendation against the feature-floor table in `modern.md`; flags any API above the floor |
| module_system | esm \| cjs \| dual | esm | Switches module guidance and import/export examples in `modern.md`; `dual` activates the dual-package hazard checks |
| browser_floor | text (e.g. "Safari 16+", "last 2 years") | none | When set, gates syntax and API advice for browser-targeted code the way `node_floor` gates Node |

Preference areas to record as the user reveals them:

- **tooling** — package manager, bundler, test runner, lint/format stack — affects which commands and config examples are offered
- **conventions** — style (semicolons, functional vs imperative iteration), error-handling shape (exceptions vs result values) — affects generated code and review verdicts
- **platform** — Deno/Bun/edge runtimes, TypeScript presence, monorepo layout — affects which floors and module advice apply
- **safety posture** — how proactively to flag legacy patterns (`var`, `==`, callback APIs) in review vs only on request

## When To Use

- Debugging JavaScript: wrong values, NaN, TypeErrors, ordering surprises, async bugs, timezone shifts
- Writing or reviewing JS (Node or browser) for correctness and modern idioms
- Choosing data structures (Object/Map/Array/Set), copy semantics, iteration strategy, or error-handling shape
- Deciding whether an ES2020+ feature is safe for the target runtime
- Diagnosing memory growth, leaks, slow code, or an unresponsive event loop
- Not for TypeScript type-system design or framework internals — this is the core language

## Quick Reference

| Situation | Go to |
|---|---|
| TypeError/ReferenceError, NaN appearing, value undefined after await, works-in-dev-only, heisenbug | `debug.md` |
| Promise/await bug, rejection handling, cancellation, concurrency limits, races | `async.md` |
| Throwing, catching, custom errors, cause chains, global error hooks, serializing errors | `errors.md` |
| `==` surprise, truthiness, implicit conversion, `??` vs `\|\|` | `coercion.md` |
| Array/Object/Map/Set choice, copying, sorting, iteration traps | `collections.md` |
| ES2020+ syntax semantics, feature floors, modules (ESM/CJS), classes, generators/iterators | `modern.md` |
| Memory grows, listener/timer leaks, WeakMap/WeakRef, heap snapshots | `memory-leaks.md` |
| Slow code, jank, benchmarks, GC pressure, workers | `performance.md` |
| Regex wrong matches, stateful `lastIndex`, catastrophic backtracking, Unicode flags | `regex.md` |
| JSON precision loss, Date/Map round-trips, reviver/replacer, canonicalization | `json.md` |
| Env vars, process exit, signals, streams, Buffer, child processes | `node.md` |
| DOM events, storage, script loading, fetch response handling | `browser.md` |
| Anything else (numbers, dates, strings, `this`, timers) | Sections below |

## Core Rules

1. `===` always; the only defensible `==` is the idiom `x == null` — it matches both null and undefined in one check. Never expand it to two comparisons.
2. Float equality is a band, not `===`: `Math.abs(a - b) <= Number.EPSILON * Math.max(1, Math.abs(a), Math.abs(b))`. Worked: `0.1 + 0.2 - 0.3` ≈ 5.6e-17, inside the band (EPSILON ≈ 2.2e-16). Money never touches floats: integer minor units.
3. Every promise gets a handler attached synchronously — a rejection with no handler when the microtask queue drains crashes Node (default since Node >=15) and fires `unhandledrejection` in browsers.
4. Mutation is opt-in: default to `toSorted`/`toReversed`/`with` and spread; `structuredClone` only when depth is real. Runtime floors for all of these: `modern.md`.
5. Durations from `performance.now()` (monotonic); wall-clock timestamps from `Date.now()`. Never subtract two `Date.now()` calls for benchmarks — NTP can step it backwards mid-measurement.
6. Sequential vs parallel is a decision you write down: `for...of` + await = sequential; `Promise.all(arr.map(f))` = parallel, fail-fast, and it does NOT cancel the losers.
7. `.length` counts UTF-16 code units, not characters: `"😀".length === 2`. Slice user-visible text with `Intl.Segmenter`, never by index.
8. Check the feature-floor table in `modern.md` before shipping ES2023+ APIs — one `toSorted` call breaks Node 18 at runtime, not at build time.

## Numbers & Money

- `Number.MAX_SAFE_INTEGER` = 2^53 − 1 = 9007199254740991 (16 digits). Integer ids with ≥16 digits can silently round in `JSON.parse` — transport ids as strings; use BigInt only for arithmetic (`JSON.stringify` on BigInt throws).
- `(1.005).toFixed(2) === "1.00"` — binary representation, not a rounding bug you can fix locally. Compute in integer cents; display through `Intl.NumberFormat`.
- Pick the parser by intent: `Number("12px")` → NaN (whole-string strict), `parseInt("12px")` → 12 (prefix). `Number("")` and `Number(null)` → 0; `Number(undefined)` → NaN.
- `["10","10","10"].map(parseInt)` → `[10, NaN, 2]`: map passes the index as radix. Always wrap: `map(s => parseInt(s, 10))`.
- `-0` exists: `Object.is(0, -0)` is false, `1 / -0 === -Infinity`. It appears when rounding negatives toward zero and survives into keys and comparisons.

## Dates & Time

- Months are 0-indexed: `new Date(2025, 0, 31)` is Jan 31.
- Parsing split: date-only ISO (`"2024-01-01"`) → UTC midnight; date-time without offset (`"2024-01-01T00:00"`) → local time. Same input day can render one day off in negative-offset zones. Any non-ISO string is implementation-defined — never parse it.
- Month math rolls over: `d = new Date(2025, 0, 31); d.setMonth(1)` → March 3 (Feb 31 normalizes). Pin the day to 1 before month arithmetic, then restore a clamped day.
- `getTimezoneOffset()` is UTC minus local: UTC+2 reports **-120**. Sign errors here produce double-offset bugs that cancel out in your own timezone and explode in others.
- Store epoch ms (`Date.now()`) plus IANA zone name; format only at the edge with `Intl.DateTimeFormat`.

## Strings & Unicode

- `length`, `slice`, `charAt` operate on UTF-16 units: `"👨‍👩‍👧".length === 8`. `[...str]` yields code points; user-perceived characters need `Intl.Segmenter`. Index-based slicing can cut a surrogate pair → U+FFFD garbage.
- Normalize before comparing user input: composed `"é"` !== `"e"` + combining accent even when rendered identically — `str.normalize("NFC")` both sides.
- Human sorting: `(a, b) => a.localeCompare(b, undefined, {numeric: true})` → `["file2", "file10"]`, accents ordered correctly. Bare `<` compares code units (`"Z" < "a"` is true).
- `/g` and `/y` regexes are stateful: `lastIndex` persists across calls, so `re.test(s)` twice on the same string can alternate true/false. Drop `/g` for single tests or reset `re.lastIndex = 0` (depth: `regex.md`).

## Objects, this & Closures

- Key order is spec'd: integer-like keys ascending FIRST, then strings in insertion order. `Object.keys({b:1, 2:2, a:3, 1:4})` → `["1","2","b","a"]`. Never encode order in numeric-string keys — use Map or an array.
- User-controlled keys on plain objects are an injection surface: `obj[userKey]` with `"__proto__"` pollutes the prototype. Use Map or `Object.create(null)` for user-keyed storage.
- `Object.hasOwn(obj, k)` over `obj.hasOwnProperty(k)`: works on null-prototype objects and can't be shadowed.
- `{...obj}` invokes getters (snapshots values); `Object.assign(target, src)` fires setters on target. Same shallow result, different side effects.
- `this`: arrow = lexical (use in callbacks); regular = call-site (methods, event handlers that want the element). `setTimeout(obj.method)` detaches `this` — wrap in an arrow.
- Closure retention is per-scope in practice: one small long-lived closure keeps alive every object referenced by ANY sibling closure of that scope. Null out large locals before returning long-lived callbacks.

## Timers & the Event Loop

- Microtasks (promise callbacks) drain completely before the next task: a self-scheduling microtask loop starves rendering and I/O forever; self-scheduling `setTimeout` yields between runs.
- Timer clamps: nested `setTimeout` beyond 5 levels → 4ms minimum; background tabs throttle to ≥1000ms (often far more). Clocks and animations must compute `elapsed = Date.now() - start` each tick — accumulating `+interval` drifts.
- Node: a `setTimeout` delay above 2147483647 ms (~24.8 days, 32-bit signed) fires almost immediately. Long schedules: persist the target time and re-arm.
- Node ordering: `process.nextTick` queue → promise microtasks → timers/macrotasks. `setImmediate` vs `setTimeout(0)`: nondeterministic at top level, `setImmediate` always first inside I/O callbacks.
- Sync work over ~50ms is a long task (the input-jank threshold); a 60Hz frame budget is 16.7ms. Chunk batch work with an awaited yield (`await new Promise(r => setTimeout(r))`) between slices.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `forEach(async x => ...)` | forEach ignores returned promises: everything runs at once, "finishes" instantly | `for...of` (sequential) or `Promise.all(arr.map(f))` |
| `Array(3).fill({})` | one shared object, three references | `Array.from({length: 3}, () => ({}))` |
| `return fetchThing()` inside try | the rejection settles after try exits — catch never sees it | `return await fetchThing()` |
| `Promise.race([op, timeout])` | the loser keeps running and holding sockets/memory | `AbortSignal.timeout(ms)` passed into the op |
| `return`/`throw` inside finally | overrides the try's result and swallows its exception | finally is for cleanup only |
| `throw "failed"` | no stack, `instanceof Error` false, breaks error middleware | `new Error("failed", {cause: err})` |
| `JSON.parse(JSON.stringify(x))` as deep clone | drops undefined/functions, Date → string, Map/Set → `{}`, throws on cycles | `structuredClone(x)` |
| `obj.fn?.()` when fn is a number | `?.` guards null/undefined only, not "not callable" | `typeof obj.fn === "function" && obj.fn()` |
| `if (x)` for "is x set" | silently drops 0, `""`, false | `x != null` |
| `setInterval` around async work | next run starts while the previous still awaits — overlapping executions | chained `setTimeout` re-armed after each completion |
| `innerHTML` with user text | the text executes as markup — XSS | `textContent`, or one sanitizer at the render boundary |

## Output Gates

Before emitting JS code or a review verdict, verify:

- Every promise created inside try is `return await`-ed, not `return`-ed?
- No API above the target floor (`modern.md` table vs `node_floor` / `browser_floor`)?
- Money in integer minor units; no ≥16-digit id passing through `JSON.parse` as a number?
- Every thrown value an Error instance, wrapped with `cause` where context was added?
- Each mutating call (`sort`, `reverse`, `splice`, `Object.assign`) intentional, not incidental?
- No plain object indexed by user-controlled keys?
- Concurrency written down: each await either deliberately sequential or batched with a bounded pool?

## Where Experts Disagree

- **Classes vs closures/factories.** Classes when many instances share methods (prototype = one function object) or brand checks matter (`#x in obj`); factories for one-off capability objects and simple DI. Field-initializer arrow functions are the worst of both (→ `modern.md` Classes).
- **Exceptions vs result values.** Throw for the unexpected: broken invariants, failed I/O you cannot proceed without. Return values for expected outcomes: validation, not-found, cancellation. The test: if the immediate caller always try/catches, it wasn't exceptional — return it.
- **Chained array methods vs loops.** Chains are the readability default; a loop wins when profiling shows intermediate-array cost or the logic needs early exit beyond `some`/`every`/`find`. Switching styles without a measurement is churn, not optimization.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/javascript (install if the user confirms):
- `typescript` — the type system layered on top of this language
- `nodejs` — Node platform operations beyond the language: servers, tooling, deployment
- `react` — framework work where these language rules get applied
- `regex` — pattern crafting beyond JS-specific regex behavior

## Feedback

- If useful, star it: https://clawic.com/skills/javascript
- Latest version: https://clawic.com/skills/javascript

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/javascript.
