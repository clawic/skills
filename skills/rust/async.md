# Async — Runtimes, `Send` Bounds, and Cancellation

An `async fn` returns a state machine that does nothing until polled. Nearly every async bug is one of three: something blocking sits on a runtime worker, a `!Send` value crosses an `.await`, or a future was dropped at a point where dropping it loses data.

## Runtime Facts That Change Your Code

- A future makes no progress unless something polls it. `let fut = do_work();` runs zero lines; only `.await`, `spawn`, or `block_on` drive it. Compiler lint `unused_must_use` catches the common form.
- `tokio::spawn` requires `Future + Send + 'static` and returns a `JoinHandle`; dropping the handle does **not** cancel the task (it detaches). `abort()` cancels it. This is the opposite of structured-concurrency defaults and the source of "why is this still running".
- The multi-threaded runtime has one worker per core by default; a task that occupies a worker without yielding starves everything scheduled on it. Tokio's own guidance is that a task should not hold a worker for more than roughly 10-100 µs of uninterrupted work.
- `Runtime::block_on` inside async code panics with "Cannot start a runtime from within a runtime". It shows up when a sync helper deep in the stack was quietly given a `block_on`.
- `#[tokio::main]` on a test does not work — `#[tokio::test]` does, and it creates a fresh current-thread runtime per test unless you ask for `flavor = "multi_thread"`.

## Blocking Is the Number One Failure

Symptom: latency climbs, CPU is low, the number of stuck requests is a small multiple of your core count.

| Kind of work | Where it belongs |
|---|---|
| File IO with `std::fs`, `std::net`, or a sync database driver | `tokio::task::spawn_blocking` |
| CPU work beyond that 10-100 µs guidance (parsing a large document, compression, crypto, image work) | `spawn_blocking`, or `rayon` with a channel back |
| `std::thread::sleep`, `Mutex::lock` under contention, `println!` to a slow terminal | Fix or move off the runtime |
| Short, bounded CPU work in a loop | Insert `tokio::task::yield_now().await` between chunks |

- `spawn_blocking`'s pool is large but finite (512 threads by default in tokio) and its tasks are **not** cancelled when the caller is dropped — a blocking call that never returns leaks a thread permanently.
- Detect it in CI rather than production: the `tokio-console` instrumentation surfaces tasks that block a worker, and a debug-build `#[tokio::main(worker_threads = 1)]` integration test turns "occasionally slow" into "deterministically hangs".

## `Send` Across `.await`

The rule the compiler applies: every value alive at an await point becomes a field of the generated state machine, so the future is `Send` only if all of them are. Offenders and fixes:

| Held across `.await` | Fix |
|---|---|
| `std::sync::MutexGuard` | Scope it in a block that ends before the await, or use `tokio::sync::Mutex` when the lock genuinely must be held across IO |
| `Rc`, `RefCell` `Ref`/`RefMut` | `Arc`, and move the mutation outside the await |
| `Box<dyn Error>` | `Box<dyn Error + Send + Sync>` |
| A `!Send` third-party handle | `tokio::task::LocalSet` plus `spawn_local`, or confine it to a dedicated thread with a channel |

The scoping idiom, which fixes most cases without changing any type:

```rust
let snapshot = { let g = state.lock().unwrap(); g.value.clone() };  // guard dies here
send(snapshot).await?;
```

Prefer `std::sync::Mutex` for short critical sections even in async code: it is faster, and the compiler enforces that you did not hold it across an await. `tokio::sync::Mutex` is for the cases where you must.

## Cancellation

Dropping a future stops it at its last await point and runs the destructors of everything it held. Nothing else happens: there is no async `Drop`, so any cleanup that itself needs `.await` will not run.

- **Cancel safety** is a property of individual futures. `AsyncReadExt::read` is cancel-safe (no bytes lost); `read_line`, `read_exact`, and most "read until" helpers are not — cancel mid-call and the partially consumed bytes are gone.
- `tokio::select!` drops every non-winning branch on each iteration. Putting a non-cancel-safe future directly in a `select!` loop silently loses data. Fix by pinning the future outside the loop (`tokio::pin!`) so the same future is polled again, or by using a cancel-safe API.
- Cleanup that must happen: do it in a `Drop` impl using only sync operations, or hand the resource to a supervisor task that owns shutdown, or use a `CancellationToken` so the task chooses its own exit point instead of being dropped.
- Timeouts are cancellation: `tokio::time::timeout(d, fut)` drops `fut` on expiry, with all the caveats above.

## Concurrency Shapes

| Want | Use | Note |
|---|---|---|
| Run N futures, need all results, same task | `futures::future::join_all` / `try_join_all` | Concurrent, not parallel — one task, one core |
| Run N futures in parallel across workers | `tokio::spawn` each, then join the handles | Each needs `Send + 'static` |
| Bounded parallelism over a stream of work | `stream::iter(items).map(f).buffer_unordered(n)` | The n is a real capacity decision; unbounded spawn is how memory disappears |
| First result wins | `select!` or `futures::future::select_ok` | Losers are dropped: cancel safety applies |
| Fan-out to many consumers | `tokio::sync::broadcast` | Slow receivers lag and get `RecvError::Lagged`, they do not block the sender |
| Backpressure between stages | `tokio::sync::mpsc` with a bounded capacity | An unbounded channel converts backpressure into an OOM |
| Anything else | Model it as tasks plus channels | Shared mutable state across tasks is the last resort, not the first |

## Async Traits

- `async fn` in traits works on stable (`rust >=1.75`), but the resulting trait is not `dyn`-compatible and the returned future has no `Send` bound — a caller that spawns it will fail to compile with a confusing message about the trait.
- For a public trait consumers will spawn: return an explicit `impl Future<Output = T> + Send`, or keep using `#[async_trait]` (which boxes, one allocation per call) and document it.
- `-> impl Future` in a trait method leaks auto traits exactly like it does in free functions: write `-> impl Future<Output = T> + Send` when callers will spawn it.

## Streams

- `Stream` is `Iterator` for async, and it is still not in std — `futures::Stream` or `tokio_stream` supply it. `.next().await` requires `StreamExt` in scope.
- A stream is lazy in the same way an iterator is, with the added trap that `while let Some(x) = stream.next().await` holds a mutable borrow of the stream for the whole loop body.
- `tokio_stream::wrappers::ReceiverStream` adapts a channel into a stream; `futures::stream::unfold` builds one from a state machine without writing a `Stream` impl by hand.

## Debug Checklist for a Stalled Async Program

1. Is the runtime alive? A dropped `Runtime` cancels every task on it — check that the handle is still owned somewhere.
2. `tokio-console` or a task dump: which task is running, which are idle, which are waiting on what.
3. Any `std::fs`, `reqwest::blocking`, sync database call, or `thread::sleep` on the async path? That is the answer more often than anything else.
4. Any unbounded channel whose receiver is slower than its sender? Memory graph rising confirms it.
5. Any `select!` loop with a non-cancel-safe branch? Data loss and stalls both come from this.
6. Single-worker reproduction: `#[tokio::main(flavor = "current_thread")]` turns intermittent starvation into a reliable hang you can attach a debugger to.
