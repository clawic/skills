# Error Handling Traps

- Classify before handling (→ SKILL.md Core Rules): operational errors (ECONNRESET, ENOENT, invalid input) get retry/degrade/report; programmer errors (TypeError, assertion) get a crash and a clean restart. A catch-all that "recovers" from both keeps corrupted state serving traffic.
- Branch on `err.code` (ENOENT, ECONNRESET, EADDRINUSE), never on `err.message` — messages change between Node versions; codes are the contract.
- Wrap without losing the root cause: `throw new Error('config load failed', { cause: err })` (node >=16.9) — chains stacks instead of flattening them into a string.
- Unhandled rejection crashes the process on node >=15 — every await chain ends in a catch or a route-level error handler.
- `process.on('uncaughtException' | 'unhandledRejection')`: log, flush, exit. After an uncaught throw, program state is unknown; these hooks are for cleanup, not recovery.
- An `'error'` event with no listener throws — every EventEmitter and every stream needs one (or use `pipeline()`, which wires all of them).
- `Promise.any` rejects with `AggregateError` — inspect `.errors`, not `.message`.
- `process.exit(1)` can truncate pending stdout/log writes — set `process.exitCode = 1` and let the loop drain; reserve `exit()` for the force-quit backstop below.

## Graceful Shutdown (order matters)

1. On SIGTERM: fail the readiness probe first, so the LB stops sending traffic.
2. `server.close()` — stop accepting, let in-flight requests finish.
3. Start an unref'd force-exit timer as backstop, shorter than the orchestrator's kill window (k8s default terminationGracePeriodSeconds is 30 s).
4. Await in-flight work, THEN close DB pools and queues — closing pools first makes step 2's surviving requests fail at the finish line.
5. Set `process.exitCode = 0` and let the loop drain.
