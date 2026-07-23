# Performance — Measure, Then Optimize

## Rules

1. Profile before touching code: `node --cpu-prof` (open the profile in DevTools) or the browser Performance panel. The hot spot is rarely where reading suggests; optimizing unprofiled code is churn (SKILL.md Where Experts Disagree).
2. Benchmarks lie three ways: JIT warmup (discard early runs), dead-code elimination (consume every result), and measuring the harness instead of the work. Compare medians over many runs on realistic payloads, timed with `performance.now()` (SKILL.md Core Rule 5).
3. The budgets live in SKILL.md Timers & the Event Loop (long-task and frame thresholds). Detect violations, don't guess: `PerformanceObserver` on `longtask` entries (browser); `perf_hooks.monitorEventLoopDelay()` (Node).

## Event-Loop Blocking

- Nothing else runs while sync work runs — one blocked tick delays every request/frame. Chief offenders: `JSON.parse`/`stringify` of multi-MB payloads (`json.md`), `*Sync` fs/zlib/crypto calls on a server, catastrophic regex (`regex.md`), long array chains over large N.
- Node: watch loop delay percentiles under load; when p99 delay approaches your latency budget, the loop itself is the bottleneck — no amount of endpoint tuning fixes it.
- Fix order, cheapest first: remove from the hot path (cache) → chunk with an awaited yield (SKILL.md Timers) → offload to a worker.

## Workers

- Workers are for CPU-bound work. I/O-bound work does not need them — async already interleaves I/O; a worker doing fetches adds cost for nothing.
- `postMessage` structured-clones the payload — cost scales with payload size, and cloning a huge object on the main thread can cost more than the work saved. Transfer `ArrayBuffer`s zero-copy: `postMessage(buf, [buf])` (the sender's copy becomes unusable, by design).
- `SharedArrayBuffer` in browsers requires cross-origin isolation (COOP + COEP response headers) — a server config prerequisite, not a code fix.
- Pool workers and reuse them; spawning per task pays startup every time. Node: `worker_threads` for CPU work, processes for isolation (`node.md` Choosing Concurrency).

## Allocation & GC Pressure

- Each link of `.map().filter().slice()` materializes a full intermediate array — for large N, one loop or lazy iterator helpers (floor: `modern.md`) do one pass with zero intermediates. Switch only on measurement (SKILL.md Where Experts Disagree).
- Keep object shapes stable: same properties, same order, no post-construction `delete` — engines fast-path monomorphic shapes; `delete` drops the object to dictionary mode. Dynamic key sets belong in a Map; "unset" is an assignment to `undefined` or null.
- Per-iteration closures and object literals in hot loops are pure allocator load — hoist what doesn't change.
- String `+=` in a loop is fine: engines use ropes, so repeated concatenation is cheap; the build-an-array-then-join ritual is legacy advice, not an optimization.

## Browser Rendering

- Layout thrashing: alternating reads (`offsetHeight`, `getBoundingClientRect`) with style writes forces a synchronous layout per round. Batch: all reads, then all writes; or read in `requestAnimationFrame` and write in the next.
- Visual updates go through `requestAnimationFrame` — timer-driven animation drifts and tears (timer clamps: SKILL.md Timers).
- Event storms (scroll, resize, pointermove): throttle to one trailing-edge run per frame, and mark scroll/touch listeners `{passive: true}` so scrolling never waits on your handler (`browser.md`).
- Animate `transform`/`opacity` — they skip layout and paint; animating `top`/`left`/`width` re-lays-out every frame.

## Caching & Memoization

- Memoize only what is pure + expensive + repeated. Two failure modes: the unbounded memo (a leak — `memory-leaks.md`) and the cold cache nobody hits (measure the hit rate; below ~50% the bookkeeping can cost more than the wins).
- Cache keys from objects: `JSON.stringify` keys differ for equal content built in different key order (`json.md` Canonicalization) — canonicalize or key on stable ids.
