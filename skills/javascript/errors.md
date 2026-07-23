# Errors — Throwing, Catching, Propagating, Reporting

## Design Rules

1. Throw Error instances only, wrapping with `cause` at every layer that adds context: `new Error("config load failed", {cause: err})` (floor: `modern.md`). A cause chain replaces the lost-context rethrow AND the string-concatenation message mangle.
2. Two kinds of failure: **operational** (expected: network down, validation, not-found) vs **programmer** (bugs: undefined access, broken invariant). Handle operational errors; let programmer errors crash loudly — a caught-and-ignored bug leaves the process in an unknown state.
3. Log OR rethrow, never both — double logging buries the real report under echoes. Exactly one boundary (the top-level handler) both logs and swallows.
4. Catch narrowly, rethrow the rest:

```js
try { await save(order); }
catch (err) {
  if (err instanceof ValidationError) return respond(400, err.message);
  throw err;   // unknown errors keep propagating
}
```

## Custom Error Classes

```js
class ConfigError extends Error {
  constructor(msg, opts) { super(msg, opts); this.name = "ConfigError"; }
}
```

- Set `this.name` explicitly — otherwise every trace reads `Error:`.
- `instanceof` fails across realms and dual-loaded packages (`modern.md` dual-package hazard) — when errors cross a boundary, brand with a `code` string property (the Node convention: `err.code === "ENOENT"`) and match on that.
- Transpiling class syntax down to ES5 breaks `instanceof` for Error subclasses — add `Object.setPrototypeOf(this, new.target.prototype)` in the constructor only if you must target ES5.

## Catching Well

- Cancellation is not a failure: check `err.name === "AbortError"` (user abort) or `"TimeoutError"` (`AbortSignal.timeout`) FIRST and return quietly — before any generic error path logs it as an outage.
- `AggregateError` (from `Promise.any`): the real reasons are in `err.errors`; the top-level message is boilerplate.
- A caught value is not guaranteed to be an Error: render with `err instanceof Error ? err.message : String(err)` before touching `.message`/`.stack`.
- Walk a cause chain: `for (let e = err; e; e = e.cause) log(e.name, e.message)`. Modern Node prints the chain in `console.error` automatically.

## Global Hooks (reporting, never control flow)

| Runtime | Hook | Semantics |
|---|---|---|
| Node | `process.on("uncaughtException")` | State possibly corrupt: log, flush, set `process.exitCode`, exit. Never resume normal work |
| Node | `process.on("unhandledRejection")` | Crashes by default (SKILL.md Core Rule 3); hook it to log before dying, not to survive |
| Browser | `window.addEventListener("error")` | Uncaught throws + resource load failures |
| Browser | `window.addEventListener("unhandledrejection")` | Promise rejections; `e.preventDefault()` suppresses the console default |

- Recovery logic belongs at the request/task boundary (one failed job ≠ dead process); global hooks are the flight recorder.
- An error thrown inside a DOM event handler or `setTimeout` callback never reaches the surrounding try/catch — it goes straight to the global hook. Wrap the callback body if you need local handling.

## Serializing Errors

- `JSON.stringify(err)` → `{}` — `message` and `stack` are non-enumerable. Serialize explicitly: `{name, message, stack, cause: err.cause && serialize(err.cause)}`.
- `structuredClone`/`postMessage` carry built-in error types with message and stack intact; custom subclasses arrive as plain `Error` — send the `code` property alongside (rule above).
- Stack format is engine-specific prose, not an API: never parse it for control flow.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `process.exit(1)` inside a catch | kills pending async including unflushed logs (`node.md`) | set `process.exitCode = 1`, let the loop drain |
| Empty `catch {}` | converts every future bug at that site into silence | at minimum: comment why + narrow the condition |
| Wrapping without `cause` | original stack and message gone; you debug the wrapper | `new Error(context, {cause: err})` |
| Matching on `err.message` text | messages change across versions and locales | match `err.name` or `err.code` |
| One try/catch around 50 lines | can't tell which operation failed, so the handler guesses | scope try to the single fallible operation |
