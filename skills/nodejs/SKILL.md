---
name: nodejs
slug: nodejs
version: 1.0.2
description: >-
  Write and debug production Node.js: event loop blocking, promise pitfalls,
  ESM/CJS interop, stream backpressure, memory leaks. Use when writing,
  reviewing, or debugging Node.js code, npm packages, or server performance.
homepage: https://clawic.com/skills/nodejs
changelog: Deeper runtime patterns and performance guidance
metadata:
  clawdbot:
    emoji: 💚
    requires:
      bins:
      - node
    os:
    - linux
    - darwin
    - win32
    displayName: NodeJS
---

## When To Use

- Writing or reviewing Node.js services, CLIs, or libraries
- Debugging a Node process: hangs, crashes, growing memory, slow endpoints, import errors
- Configuring npm installs, lockfiles, dependency updates, or publishing
- Diagnosing production incidents: intermittent 502s behind a load balancer, event loop stalls, leaked descriptors
- Not for browser JavaScript or TypeScript type design — Node runtime concerns only

## Quick Reference

| Situation | File |
|---|---|
| Promise pitfalls, event loop ordering, callback interop | `async.md` |
| require vs import, ESM/CJS interop, dual packages | `modules.md` |
| Crashes, unhandled rejections, graceful shutdown | `errors.md` |
| Streams, backpressure, pipeline | `streams.md` |
| Memory leaks, blocked event loop, worker threads | `performance.md` |
| Injection, path traversal, secrets, prototype pollution | `security.md` |
| Flaky tests, mocking, fake timers | `testing.md` |
| npm versions, lockfiles, audit, publishing | `packages.md` |
| Anything else Node-specific | Core Rules and Traps below first, then the closest file above |

## Core Rules

1. **Budget sync work at ~10 ms per slice.** Max throughput per process ≈ 1000 ms ÷ block_ms: a 50 ms sync `JSON.parse` in a request handler caps that process near 20 req/s, and every concurrent request inherits the stall. Repeatedly over budget → `worker_threads`; one-off → partition with `setImmediate` (→ `async.md`).
2. **Handle operational errors, crash on programmer errors** (Joyent doctrine). ECONNRESET, ENOENT, bad user input → handle at the call site. TypeError, undefined property → let it crash; the supervisor restarts a clean process. Catch-all recovery keeps corrupted state alive.
3. **"Async" fs, dns.lookup, crypto, and zlib share 4 libuv threads** (default `UV_THREADPOOL_SIZE`; max 1024). The 5th concurrent `fs.promises.readFile` queues behind the first 4. Slow "async" crypto/fs under load → raise the pool size or move the work off the process.
4. **Cap concurrency explicitly.** `Promise.all(items.map(fetch))` over 10k items opens 10k sockets — EMFILE at the common 1024 fd soft limit. Batch or use a limiter; 8-32 concurrent per upstream host is a sane starting cap, raise on evidence.
5. **`keepAliveTimeout` must exceed the load balancer's idle timeout.** Node's default is 5 s; ALB's default idle is 60 s, so Node closes sockets the LB just reused → intermittent 502s. Set keepAliveTimeout = LB idle + 5 s, and `headersTimeout` just above that (ALB: 65 s / 66 s).
6. **Exact versions for apps, ranges for libraries; `npm ci` in anything automated.** `npm install` in CI can rewrite the very lockfile you meant to test (→ `packages.md`).
7. **One process per container.** Let the orchestrator scale replicas; `cluster` only on bare metal/VMs you own entirely — two schedulers (k8s + cluster) fighting obscures both.

## Output Gates

Before shipping Node code, check:

- Any `*Sync` call, large `JSON.parse`, or unbounded loop reachable from a request handler? (Rule 1)
- Every stream wired through `pipeline()` or carrying its own `error` handler? An unhandled stream `error` is a process crash.
- Any `Promise.all` over input the user controls the size of? (Rule 4)
- All `process.env` reads parsed and validated once at startup — they are strings, possibly undefined?
- Shutdown path exists: SIGTERM → stop accepting → drain → exit, in that order (→ `errors.md`)?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Unhandled promise rejection | Crashes the process on node >=15 | `.catch()` or try/catch on every await chain |
| `.pipe()` without error wiring | Errors don't cross pipes; leaks then crashes | `stream.pipeline()` |
| Forgotten `await` inside try/catch | Rejection skips the catch, surfaces later or never | Lint with `no-floating-promises` |
| `exports = x` in CJS | Rebinds a local alias; `module.exports` unchanged | `module.exports = x` |
| `__dirname` in ESM | Doesn't exist there | `import.meta.dirname` (node >=20.11), else `fileURLToPath(import.meta.url)` |
| Mutating a `require()`d object | Module cache: every consumer sees the mutation | Export factories, or freeze exports |
| `process.env.PORT` used as a number | Env values are strings (`"3000"`) or undefined | Parse and validate env once at startup |
| Listeners added per request | Accumulate until MaxListenersExceededWarning — a leak signal, not a limit | `once()`, or remove on completion |
| Throwing non-Error values | No stack; `instanceof Error` checks fail | `throw new Error(msg, { cause })` |
| `node:vm` as a sandbox | Not a security boundary — its own docs say so | Isolate untrusted code at process/container level |

## Where Experts Disagree

- **Crash-and-restart vs catch-and-continue**: crash when a supervisor restarts you and in-process state may be corrupt; continue only for known operational errors scoped to a single request.
- **Dual CJS+ESM packages vs ESM-only**: dual carries the two-instance hazard (`instanceof` fails across copies); ESM-only is viable once consumers are ESM themselves or run node >=22.12 (`require()` of sync ESM works).
- **Jest vs `node:test`**: Jest earns its weight on existing suites and rich matcher/snapshot needs; `node:test` (stable node >=20) for new libraries — zero transform layer, native ESM, no hoisting magic.

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/nodejs.
