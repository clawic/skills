# Async Traps

## Event Loop
- Phase order per turn: timers → poll (I/O) → check (`setImmediate`) → close. `process.nextTick` and microtasks drain between every phase — so recursive `nextTick` starves I/O forever, recursive `setImmediate` doesn't (one per turn).
- Partition long sync loops with `await setImmediate()` from `node:timers/promises` roughly every ~10 ms of work (budget → SKILL.md Core Rules). `nextTick` in a loop does NOT yield to I/O.
- `setTimeout(fn, 0)` vs `setImmediate`: inside an I/O callback, setImmediate always runs first; outside I/O their order is nondeterministic — never depend on it.
- Deferring from a cache hit: use `queueMicrotask`, not `nextTick` — same effect, no starvation footgun, portable.

## Promises
- `Promise.all` fails fast but does NOT cancel the losers: they keep running, and their later rejections become unhandled (crash on node >=15). Attach a no-op `.catch` to survivors or thread an AbortSignal through.
- Same with `Promise.race` timeouts: the slow branch keeps consuming resources. Use `AbortSignal.timeout(ms)` (node >=17.3) so the operation actually stops.
- Sequential `await` in a loop is a decision, not automatically a bug: keep it when order or rate limits matter; otherwise `Promise.all` over capped batches (cap → SKILL.md Core Rules).
- `return await` is redundant — except inside try/catch, where dropping the await makes the rejection skip your catch.
- An `async` function that throws before its first `await` still returns a rejected promise. A plain function that returns a promise but throws first throws synchronously — callers that only `.catch()` miss it. Invisible until it isn't.
- Forgotten `await` inside try/catch: the catch never fires; the rejection surfaces as unhandledRejection long after the stack that caused it is gone. Lint: `no-floating-promises`.
- Top-level `await` in ESM blocks every module that imports yours — acceptable for app entry/config, poison in a library.

## Callbacks
- Callback errors don't throw — check `if (err)`; try/catch around the call catches nothing.
- Zalgo: an API that calls back synchronously on cache hits and asynchronously otherwise reorders caller logic unpredictably — always defer (`queueMicrotask`) when answering from cache.
- Wrapping callback APIs by hand loses the error argument sooner or later — `util.promisify` respects the `(err, value)` contract; write custom wrappers only for non-standard signatures.
