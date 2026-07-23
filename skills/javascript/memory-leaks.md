# Memory — Leaks, Growth, Diagnosis

## Is It Actually a Leak?

1. A sawtooth is normal: V8 delays collection while there is headroom, so growth alone proves nothing. A leak = the post-GC baseline rising across cycles.
2. Node: `process.memoryUsage()` — `heapUsed` is JS objects; a growing `rss` with flat `heapUsed` means native memory (Buffers count as `external`, addons, fragmentation) — heap snapshots will NOT show that; audit Buffer lifetimes and native deps instead.
3. Prove it in a test: run with `--expose-gc`, call `global.gc()`, record `heapUsed`, repeat the suspect operation N times, `global.gc()` again — growth linear in N is the confession; constant offset is warmup.

## The Usual Suspects (ranked)

| Leak | Mechanism | Fix |
|---|---|---|
| Unbounded cache | a Map/object that only ever gains entries | LRU with a max size; WeakMap when keys are objects you don't own |
| Listeners never removed | the emitter/DOM node retains every handler + its closure | `{signal}`-grouped listeners (`browser.md`), `removeEventListener`/`removeListener`, `{once: true}` |
| `setInterval` never cleared | callback + closure alive forever (and overlapping — SKILL.md Traps) | `clearInterval` on teardown; chained `setTimeout` |
| Closure capturing a large scope | sibling-closure retention (SKILL.md Objects, this & Closures) | null out large locals before returning long-lived callbacks |
| Detached DOM nodes | a JS reference keeps a removed subtree alive | drop refs on removal; find them via the snapshot "Detached" filter |
| Never-settling promises | every awaiter and its scope pinned forever | timeout every external wait (`async.md` Timeouts) |
| Module-level accumulators | module scope is process-lifetime scope | move collections to request/task scope |

- Node's `MaxListenersExceededWarning` (default threshold: 10 listeners per event) is a leak detector, not a nuisance — raising the limit without identifying the accumulating registration just silences the alarm.

## Heap Snapshots — the 3-Snapshot Method

1. Snapshot at steady state → perform the suspect action a known N times (say 10) → snapshot → N more → snapshot.
2. Compare: snapshot 3 filtered to "objects allocated between snapshots 1 and 2" — real leaks appear in multiples of N; noise doesn't.
3. Sort by retained size; the Retainers pane answers the only question that matters: *what still points at this?* Fix the pointer, not the object.
4. Tooling: DevTools Memory tab (browser); `node --inspect` + `chrome://inspect` (Node). Non-interactive: `v8.writeHeapSnapshot()`, or start with `--heapsnapshot-signal=SIGUSR2` and `kill -USR2 <pid>` on the live process.

## Weak References

- WeakMap/WeakSet: metadata keyed on objects you don't own — the entry dies with the key. Not iterable by design (iteration would observe GC).
- A Map with object keys is the accidental strong version of a WeakMap — the most common cache leak, and a one-line fix.
- `WeakRef` + `FinalizationRegistry`: last resort, for opportunistic caches of recomputable values only. GC timing is nondeterministic across engines — never hang correctness on finalization; it may run late or never.

## Ceilings & Containers

- Set an explicit `--max-old-space-size` (MB) when memory is bounded; in a container put it at ~75-80% of the container limit, leaving room for buffers and native memory.
- If the heap cap sits ABOVE the container limit, the kernel OOM-kills first: exit 137 and no JS stack, no heap dump — the JS-level `heap out of memory` abort (which prints a trace) only happens when V8's own cap is the one that's hit. Container-side diagnosis: the `docker` skill.
