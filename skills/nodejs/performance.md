# Performance Traps

## Blocked Event Loop
- Symptom signature: everything slows at once and one core pins at 100%. Confirm with `perf_hooks.monitorEventLoopDelay()` — sustained delay above your sync budget (→ SKILL.md Core Rules) means the loop is blocked; flat delay with slow responses means the bottleneck is downstream.
- Usual blockers: `*Sync` fs/crypto calls, large `JSON.parse`/`stringify` (tens of MB → hundreds of ms on the loop), catastrophic regex backtracking (`/(a+)+$/` on `"aaaa…X"` hangs), tight loops over big arrays.
- Locate it: `node --inspect`, record a CPU profile under real load, look for the widest self-time frames — not the deepest stacks.
- `console.log` to a terminal or file is synchronous on POSIX — hot-path logging blocks the loop; use an async structured logger (pino) in request paths.
- Parallel-looking fs/dns.lookup/zlib/crypto work serializes on 4 libuv threads (→ SKILL.md Core Rules) — "async" is not "concurrent" past the pool size.

## Memory Leaks — diagnosis order
1. Sample `process.memoryUsage()` over time. `heapUsed` climbing across GC cycles = JS-object leak. RSS climbing while heapUsed stays flat = Buffers/native memory/fragmentation — a different hunt; heap snapshots won't show it.
2. Take two heap snapshots (`--inspect` → DevTools Memory) minutes apart under load; sort the comparison by retained-size delta per constructor.
3. Usual suspects: module-level Map/array caches (bound them — LRU with a max), listener arrays (EventEmitter warns past 10 listeners on one event; raising the limit hides the leak, never fixes it), `setInterval` closures never cleared, promise chains that never settle.
- Containers: the default heap cap varies by version and available memory — set `--max-old-space-size` to ~75% of the container limit. RSS = heap + buffers + stacks + native, so heap = limit guarantees OOMKill.

## Worker Threads
- Spawning a worker costs milliseconds and megabytes — pool them (piscina); never spawn per request.
- `postMessage` structured-clones by default: copying a 100 MB buffer per task erases the parallelism you bought. Pass ArrayBuffers in the transfer list (zero-copy; sender loses access) or share via SharedArrayBuffer.
- Workers pay off for CPU-bound work only — I/O concurrency already belongs to the event loop; putting it on workers adds copying for nothing.
