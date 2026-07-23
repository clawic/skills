# Debugging — Symptom to Cause in Minutes

Work symptom-first. Each chain is ordered by probability; every step is a check, not a guess.

## The Universal First Three

1. Read the FIRST error, not the loudest — later errors are usually fallout from the first one.
2. Log the value AND its type at the failure site: `console.log({x, t: typeof x})` — half of all TypeErrors are the wrong type arriving, not wrong logic.
3. Reproduce minimally: extract the failing expression into a scratch file/console with hardcoded input. If it passes there, the bug is in what feeds it, not in it.

## "Cannot read properties of undefined" / "x is not a function"

First split by error class: **ReferenceError** = the NAME doesn't exist (typo, wrong scope, TDZ — `let`/`const`/class used before its declaration line); **TypeError** = the name resolved but the VALUE is wrong. The chains below are the TypeError side.

1. Identify WHICH link is undefined: for `a.b.c.d`, log each link. `?.` silences the throw without answering the question — diagnose first, guard after.
2. Trace where the undefined came from:

| Origin | Signal | Fix |
|---|---|---|
| async data not loaded yet | works on retry / after refresh | gate use on loaded state, not on time |
| detached `this` | method passed as a callback | arrow wrapper (SKILL.md Objects, this & Closures) |
| circular import | undefined only at module top level, fine when called later | `modern.md` Modules |
| CJS/ESM interop | `.default` shows up in logs, `x.default is not a function` | fix the import form (`modern.md`) |
| API above the runtime floor | works locally, throws in prod | `modern.md` feature floors |
| misspelled / wrong-case key | `Object.keys(obj)` shows a near-miss | log the keys, not the value |

## NaN Appearing

- NaN is viral — any arithmetic touching it stays NaN. Hunt the FIRST NaN, not the place you noticed it.
- Sources, ranked: `undefined` in arithmetic (missing property is #1) · `Number("")`/`parseInt` semantics (SKILL.md Numbers & Money) · `.getTime()` on an Invalid Date · object coerced through `toString`.
- Instrument the data boundary: `if (Number.isNaN(v)) throw new Error(...)` where values enter. `Number.isNaN` only — global `isNaN` coerces first (`isNaN("foo")` → true even though `"foo"` is not NaN).
- Invalid Date is the stealth NaN factory: `new Date(bad).getTime()` → NaN, then every duration downstream is NaN. Validate with `Number.isNaN(d.getTime())` right after parsing.

## Wrong Value After Await

- `undefined` after await → the awaited function has no `return` on some path (classic: a `.then` chain whose last callback returns nothing).
- `[object Promise]` in output, or `if (check())` passing for everyone → missing `await`; promises are always truthy. Grep the call sites of every async function.
- Value is stale → await-torn state; guard with a version stamp (`async.md`).
- Correct sometimes, wrong under load → race between uncoordinated async writers; serialize (`async.md`).

## Works in Dev, Fails in Prod

Check in this order; each is a one-minute test:

| Difference | Check |
|---|---|
| Runtime version below a feature floor | `modern.md` table vs the prod runtime; the failure is a runtime TypeError |
| Minification renamed identifiers | code relying on `fn.name` / `constructor.name` / error-message matching |
| `NODE_ENV=production` branches | libraries strip validation and error detail in prod mode |
| Case-sensitive filesystem (Linux) | import path differing only by case from the file on disk |
| Server TZ (UTC) vs laptop local | date-only ISO parsing shifts a day (SKILL.md Dates & Time) |
| Env var present locally only | log presence (`"KEY" in process.env`), never values |
| Bundler tree-shook a side-effect import | re-import explicitly for effect; check `sideEffects` config |

## Heisenbugs (vanishes when you look)

- DevTools logs objects as LIVE references: the console shows the state at expansion time, not at log time. Snapshot it: `console.log(structuredClone(x))`.
- Adding logs or a breakpoint changes timing → it's a race; stop adding logs and apply `async.md` Await-Torn State.
- Works when stepping, fails when running = an ordering assumption with no `await` enforcing it. Make the ordering explicit.

## Reading Stack Traces

- Async/await preserves async stacks in modern engines; long `.then` chains lose frames — one more reason to prefer `await`.
- `throw "string"` → no stack at all (SKILL.md Traps). Errors crossing worker/process boundaries lose subclass identity — serialize deliberately (`errors.md`).
- Production traces without source maps point at bundle lines, not your code: retain the maps at build time or the trace is decoration.
- Node: `Error.captureStackTrace(err, factoryFn)` hides the factory's own frames from consumers.

## When You Are Truly Stuck

Bisect, don't stare: git bisect for regressions; input bisection for data bugs (halve the payload until the minimal failing record remains); comment-out bisection for pipelines. Each halving is one run — 1,000 suspects fall in 10 runs.
