# Exceptions — Throwing, Catching, Closing, Retrying

Design principle: an exception is a message to the code that can DO something about it. If no caller can act, the only useful behaviors are fail fast and log with full context.

## Choosing the Type

| Situation | Throw |
|---|---|
| Caller passed something illegal | `IllegalArgumentException` (or `NullPointerException` via `Objects.requireNonNull`) |
| Object is in the wrong state for this call | `IllegalStateException` |
| Feature genuinely not implemented | `UnsupportedOperationException` |
| Domain rule violated, caller may recover | Your own unchecked exception, one per domain concept |
| Recoverable and the caller has a real alternative | A checked exception — rare, and deliberate |
| Anything the JVM owns (`OutOfMemoryError`, `StackOverflowError`) | Nothing: do not catch `Error` |

- One exception type per *decision the caller makes*, not per error message. If every caller catches all your types identically, you have too many.
- Include the offending value in the message: `"orderId=" + id + " not found"` beats "not found" by an hour of debugging. Never include secrets or full payloads (`security.md`).
- Preserve the cause: `throw new StorageException("saving order " + id, e)`. A new exception without the cause deletes the only useful part of the trace.
- Do not use exceptions for control flow — a stack trace capture costs orders of magnitude more than a boolean return. The escape hatch, for a genuinely hot signalling path, is a preallocated exception with `super(msg, null, false, false)` (no suppression, no writable stack trace).

## Catching Well

- Catch the narrowest type you can act on. `catch (Exception e)` at a low level swallows programming errors and `InterruptedException` alike (SKILL.md Traps).
- Multi-catch keeps the shape: `catch (IOException | TimeoutException e)`; the variable is implicitly final.
- Catch-log-rethrow logs the same failure at every layer. Pick one place — the request boundary — and log there once, with the full chain.
- `catch (Throwable t)` is defensible in exactly one place: the top-level loop of a worker that must not die. Rethrow `Error` after recording it; a JVM in an `OutOfMemoryError` state cannot be reasoned about.
- An empty catch block needs a comment explaining the impossibility, or it is a bug waiting for its incident.

## finally and try-with-resources

- `return` inside `finally` discards the exception in flight and any earlier `return` — a silent swallow the compiler warns about only with lint enabled. Never return from `finally`.
- `finally` also runs on `break`, `continue`, and `return`; it does not run on `System.exit` or a JVM crash.
- try-with-resources closes in reverse declaration order and turns a `close()` failure into a suppressed exception attached to the primary one (`e.getSuppressed()`). Hand-written `finally { close(); }` does the opposite: the close failure REPLACES the real exception.
- Multiple resources in one statement: `try (var conn = ...; var stmt = conn.prepareStatement(sql))`. Each later resource sees the earlier ones.
- An effectively-final existing resource can be used directly (9+): `try (existing) { ... }`.
- Wrapping streams: closing the outermost closes the chain. Closing a `ByteArrayOutputStream` is a no-op, but closing a `BufferedWriter` is what actually flushes — a "missing data" bug that only appears for small files.

## Interrupts (the most-broken contract in Java)

- `InterruptedException` means "someone asked you to stop". Catching it clears the interrupt flag, so ignoring it makes the request invisible to everyone above you.
- Two correct responses: propagate it (declare `throws InterruptedException`), or restore the flag and return (SKILL.md rule 3).
- In a loop, also check `Thread.currentThread().isInterrupted()` between iterations: long CPU-bound work is never interrupted automatically.
- `Future.cancel(true)` interrupts; `CompletableFuture.cancel(true)` does not touch the running thread (`async.md`).
- Blocking I/O on a classic `InputStream` is not interruptible at all. Use timeouts on the socket, or `InterruptibleChannel`, or accept that shutdown waits.

## Stack Traces Worth Reading

- The FIRST `Caused by:` from the bottom is the origin; the top of the trace is where it surfaced (`debug.md`).
- `... 47 more` means those frames are identical to the enclosing trace — not lost.
- Async and reactive stacks lose the caller's frames by construction. Attach context in the message or an MDC key, because the trace will not carry it.
- Lost stack traces: rethrowing `throw e.getCause()` discards the wrapper's frames; `e.printStackTrace()` into a swallowed stream discards everything.
- No stack trace at all on a repeated exception: JIT fast-throw (SKILL.md Exception Triage).

## Retry and Recovery

- Retry only idempotent operations, only on transient failures. The full backoff/jitter and timeout-budget rules live in `async.md`.
- Distinguish three outcomes explicitly: succeeded, failed-retryable, failed-permanent. Code that treats all failures alike either hammers a dead service or gives up on a blip.
- A fallback value must be honest: returning an empty list on a timeout tells the caller "no data exists" when the truth is "unknown". Prefer failing with a clear type over lying quietly.
- Log at the level that matches the action: `WARN` when you recovered, `ERROR` when a human must act, `INFO` for expected outcomes. An `ERROR` per retry attempt trains everyone to ignore ERROR.

## Checked Exceptions in Modern Code

- They do not pass through lambdas, so any checked exception in a stream forces a wrapper — one reason libraries moved to unchecked. The three honest workarounds are in `lambdas.md`.
- The common wrapper: `throw new UncheckedIOException(e)` for `IOException`, which keeps the type recoverable at the boundary.
- If you own the API, sealing the failure into a return type (a sealed `Result` hierarchy, `classes.md`) is a legitimate third option — but never mix it with exceptions in the same layer.
- The frontier is in SKILL.md, Where Experts Disagree.
