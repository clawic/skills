# Async — CompletableFuture, Executors, Timeouts, Cancellation

This file is about composing asynchronous *results*. Guarding shared mutable *state* is `concurrency.md`.

## Which Executor Runs Your Callback

- `supplyAsync(fn)` without an executor runs on `ForkJoinPool.commonPool` — sized `availableProcessors() − 1`, shared with parallel streams across the whole JVM. **Never do blocking I/O there**: a handful of blocked tasks starves every parallel stream in the process, and in a 1-CPU container parallelism is 0 (SKILL.md rule 9).
- Always pass your own executor for I/O: `supplyAsync(this::callApi, ioExecutor)`.
- Non-`Async` stages (`thenApply`) run on **whichever thread completed the previous stage** — or on the calling thread if the future was already complete. That is why the same code runs on a different thread depending on timing.
- `thenApplyAsync(fn)` with no executor moves the work back to the common pool. Pass the executor explicitly in every `*Async` call, or the pool choice becomes an accident.

## The Core Distinctions

| Pair | Difference | Pick |
|---|---|---|
| `thenApply` vs `thenCompose` | `thenApply(f)` where `f` returns a future gives you `CompletableFuture<CompletableFuture<T>>`; `thenCompose` flattens it | `thenCompose` whenever the function is itself async — it is `flatMap` |
| `thenCombine` vs `thenCompose` | Combine merges two INDEPENDENT futures running in parallel; compose chains a dependent one | Combine for fan-out, compose for sequencing |
| `join()` vs `get()` | `join` throws unchecked `CompletionException`; `get` throws checked `ExecutionException` + `InterruptedException` | `join` inside lambdas, `get(timeout)` at the boundary |
| `exceptionally` vs `handle` vs `whenComplete` | `exceptionally` recovers with a value; `handle` transforms both paths into a new value; `whenComplete` observes both and passes the original result through | Recover with `exceptionally`, log with `whenComplete` |
| `allOf` vs `anyOf` | `allOf` returns `CompletableFuture<Void>` (results must be re-read from each future); `anyOf` returns `CompletableFuture<Object>` and loses the type | Use `allOf(...).thenApply(v -> futures.stream().map(CompletableFuture::join).toList())` |

- Exceptions in a stage wrap into `CompletionException`. Unwrap before matching: `ex instanceof CompletionException ? ex.getCause() : ex`. Code that catches your own exception type and never fires is almost always missing this.
- A `CompletableFuture` you never attach a terminal handler to swallows its failure silently — there is no unhandled-rejection warning as in other languages. Every chain ends in `whenComplete` or is `join`ed.

## Timeouts

- `orTimeout(3, SECONDS)` fails the future with `TimeoutException`; `completeOnTimeout(fallback, 3, SECONDS)` completes it with a default (both JDK 9+).
- **A timeout does not cancel the work.** The HTTP call, the query, the thread all keep running. Set the timeout in the underlying client too (connect + read), or you have added latency control without freeing the resource.
- Timeout budget for a call chain: the caller's timeout must exceed the sum of the downstream ones, or you time out upstream while a healthy retry is still in flight. Worked: 3 attempts × 2s read timeout + 2 × 200ms backoff = 6.4s, so the caller's budget is ≥7s or the retry policy must shrink.

## Cancellation

- `future.cancel(true)` on a `CompletableFuture` only completes it exceptionally with `CancellationException`; unlike a `FutureTask`, it does **not** interrupt the running thread.
- To make work actually stoppable: run it through an `ExecutorService`, hold the returned `Future`, call `cancel(true)`, and make the task check `Thread.currentThread().isInterrupted()` between steps and honour `InterruptedException` (SKILL.md rule 3).
- Cancellation does not propagate downstream in a chain of futures: cancelling the last stage does not stop the first one. If you need that, keep the handle to the root or use structured concurrency.

## Structured Concurrency (preview)

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {   // --enable-preview
    var user  = scope.fork(() -> fetchUser(id));
    var order = scope.fork(() -> fetchOrders(id));
    scope.join().throwIfFailed();
    return new Page(user.get(), order.get());
}
```

- The model: subtasks cannot outlive the block, the first failure cancels the siblings, and the stack trace stays readable. It removes the leaked-task and forgotten-cancellation categories entirely.
- Still a **preview API through JDK 25** — it needs `--enable-preview`, the API changed between previews, and preview classes are not binary-compatible across releases. Gate it behind the `preview_features` variable (SKILL.md Configuration).
- Until then, the closest stable equivalent is `invokeAll` on a virtual-thread executor with a try-with-resources block, which at least bounds the lifetime.

## Blocking Correctly

- `CompletableFuture.join()` on a platform thread blocks that thread. On a **virtual** thread it is cheap — which is why virtual threads plus plain blocking calls now beat most hand-rolled async pipelines for readability at similar throughput (`concurrency.md`).
- Never `join()` inside a stage that runs on the pool executing that same future — the classic self-deadlock (`debug.md`).
- `CompletableFuture.supplyAsync(...).join()` immediately is a thread hop for nothing: call the method directly.

## Retries That Do Not Make Outages Worse

- Retry only idempotent operations, only on retryable failures (timeouts, 5xx, connection resets) — never on a 4xx or a validation error.
- Exponential backoff **with jitter**: `sleep = random(0, base × 2^attempt)`, capped. Worked with base 100ms and cap 2s: attempts wait somewhere in 0-100ms, 0-200ms, 0-400ms... Without jitter, every client retries in lockstep and re-creates the spike that caused the failure.
- Cap total attempts AND total elapsed time; the elapsed cap is what protects the caller's own timeout budget.
- A retry loop wrapped around a call that has no timeout is a hang with extra steps.
- Above a few call sites, a circuit breaker beats per-call retries: stop calling a service that is failing, so it can recover.

## Sanity Checks Before Shipping an Async Chain

- Does every chain end in a terminal handler that logs failures?
- Is a non-default executor passed to every stage that blocks?
- Does each external call have its own timeout, independent of `orTimeout`?
- Are `CompletionException`s unwrapped before type-matching?
- Is the thread-name prefix distinctive enough to identify this work in a thread dump?
