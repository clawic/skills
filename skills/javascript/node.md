# Node — Process, Env, Streams, Buffers, Children

Language-level rules live in SKILL.md; this file is the Node platform surface (event-loop ordering: SKILL.md Timers; thread-pool queueing: `async.md`).

## Env & Config

- Every `process.env` value is a string or `undefined`: `"false"` and `"0"` are truthy. Parse explicitly — `env.FLAG === "true"` for booleans, `Number(env.PORT)` with an NaN check for numbers.
- Native env files: `node --env-file=.env` (Node >=20.6) — dotenv without the dependency. Load-order trap either way: config read at module top level sees only variables set before that module was imported.
- Mutating `process.env` affects children spawned afterwards, never processes already running.

## Exit & Signals

- `process.exit()` aborts pending async work INCLUDING unflushed stdout when it's a pipe — the log line explaining the crash is exactly what gets lost. Set `process.exitCode = 1` and let the loop drain.
- A Node process exits when nothing keeps the loop alive; servers, sockets, and timers hold it open. A process that won't exit has a live handle — `.unref()` timers/watchers that shouldn't hold the process, close what should be closed.
- Graceful shutdown: handle `SIGTERM` + `SIGINT`; once you register a handler, the default die-on-signal is gone and exiting becomes YOUR job — stop accepting (`server.close`), finish in-flight work under a deadline, then exit. Container orchestrators send SIGTERM and SIGKILL after a grace period (the `docker` skill covers that side); a handler that never exits is a container that hangs its full stop-timeout.
- `SIGKILL`/`SIGSTOP` cannot be caught — there is no cleanup path; design state to survive them (atomic writes below).

## Streams & Backpressure

- `pipeline(src, ...transforms, dest)` from `stream/promises` over `.pipe()`: it propagates errors and destroys every stage; `.pipe()` leaks the source when the destination errors.
- Respect backpressure: `write()` returning `false` means pause until `'drain'`. Ignoring it buffers unboundedly — the "streaming" server that OOMs anyway.
- Consume readables with `for await (const chunk of readable)`. Chunk boundaries are arbitrary: never assume one chunk = one line/record — split on delimiters (`readline` for lines).
- Multi-byte characters split across chunk boundaries corrupt per-chunk `toString()` — decode with `new TextDecoder("utf-8")` + `{stream: true}` (or `string_decoder`), or decode once after concatenation.

## Buffer

- `Buffer.allocUnsafe` returns uninitialized memory — recycled process memory can leak into output. `Buffer.alloc` unless profiling shows the zero-fill matters AND you overwrite every byte.
- `buf.length` is bytes; `string.length` is UTF-16 units (SKILL.md Core Rule 7). Byte length of a string is `Buffer.byteLength(s, "utf8")` — miscomputed `Content-Length` headers live here.
- Compare secrets with `crypto.timingSafeEqual`, never `===` or `buf.equals` — equality that exits early leaks position information.

## Child Processes

| API | Shell | Output | Use |
|---|---|---|---|
| `execFile` | no | buffered | run a binary with args — the default choice |
| `spawn` | no | streaming | long-running or large-output commands |
| `exec` | yes | buffered | shell features needed; NEVER with interpolated input |
| `fork` | no | + IPC channel | Node-to-Node with message passing |

- Anything with a shell (`exec`, or `{shell: true}` on spawn) turns arguments into one parsed command line — user input there is command injection. The no-shell APIs pass args as an array, uninterpreted.
- Buffered APIs kill the child when output exceeds `maxBuffer` (default 1 MiB) — chatty commands need `spawn`.
- Handle BOTH `'error'` (spawn failure: ENOENT, permissions) and non-zero exit codes — they arrive through different channels.
- A child whose stdout pipe fills up and is never read BLOCKS mid-write: consume or explicitly ignore stdio.

## fs

- Don't `existsSync` then read — TOCTOU race plus a wasted syscall. Read and catch `err.code === "ENOENT"`.
- Error codes worth recognizing on sight: `ENOENT` missing path · `EACCES` permissions · `EADDRINUSE` port taken (often the previous instance still dying) · `ECONNRESET` peer dropped mid-conversation · `EPIPE` wrote to a closed pipe/socket · `EMFILE` fd exhaustion (bound your concurrency — `async.md` pool).
- Atomic writes: write to a temp file, then `rename` onto the target (same filesystem) — readers and a mid-write crash never see a half-written file.

## Choosing Concurrency

- I/O-bound: plain async — already concurrent, nothing else needed. CPU-bound JS: `worker_threads` (`performance.md` Workers). Scale a server across cores: multiple processes behind a load balancer (or `cluster`). Run other programs: `child_process`. Picking workers for I/O is the common miss — it adds serialization cost and saves nothing.
