# Async: Ordering, Rejections, Concurrency

One mental picture: call stack empties → (Node) `process.nextTick` queue → ALL microtasks (promise callbacks) → one macrotask (timer/I-O) → repeat. `await` = suspend now, resume as a microtask.

## Unhandled Rejections

- Attach `.catch`/try-catch in the same synchronous block that creates the promise (why: → SKILL.md Core Rule 3). Attaching in a later tick can fire the unhandled event before your handler exists.
- Fire-and-forget must be explicit: `void op().catch(log)` — the `void` marks intent for readers and linters.
- A promise created inside `setTimeout`/event callbacks is detached from any outer chain: its rejection escapes your surrounding try/catch entirely. Give it its own catch.
- `new Promise(async (resolve) => ...)`: a throw inside the async executor rejects the executor's own promise, not yours → silently swallowed. Never mark an executor async; around await-able code you rarely need `new Promise` at all.
- Top-level await that rejects fails the whole module graph — every importer errors, not just yours.

## try/catch Boundaries

- Inside try/catch/finally, `return promise` lets the rejection escape; only `return await promise` routes it through THIS catch. Outside any try, plain `return promise` is fine.
- The old advice that `return await` costs an extra tick is obsolete in modern V8 — prefer it inside try for correctness and better stack traces.

## Parallelism & Cancellation

- `Promise.all` is fail-fast for your code only: the losing promises keep executing to completion. Cancellation is cooperative — thread one `AbortController` through and pass/check its signal.
- `Promise.allSettled` → `{status, value|reason}` per item: use for independent batches where partial success is the normal case.
- `Promise.any`: first fulfillment wins; if all reject you get `AggregateError` with `.errors` — the fallback/mirror-request primitive.
- Cap fan-out: browsers open ~6 HTTP/1.1 connections per origin, so `Promise.all` over 200 fetches queues anyway while pinning memory; on servers, unbounded fan-out over thousands of items exhausts sockets and file descriptors. Pool pattern:

```js
async function pool(items, n, fn) {
  const results = []; let i = 0;
  await Promise.all(Array.from({length: Math.min(n, items.length)}, async () => {
    while (i < items.length) { const idx = i++; results[idx] = await fn(items[idx]); }
  }));
  return results;
}
```

- Pre-created promise array + sequential awaits: a later promise can reject while you still await an earlier one → unhandled-rejection window. Either `Promise.all` the array, or create each promise lazily inside the loop.
- Node: fs/dns/crypto callbacks run on a libuv pool of 4 threads by default (`UV_THREADPOOL_SIZE`) — "async" fs calls still queue behind each other under load.

## Await-Torn State

- Every `await` is a yield point: anything read before it can be stale after. Check-then-act across an await is a race (read cache → await fetch → write cache clobbers a newer writer).
- Guard with a version stamp: `const v = state.version; const data = await load(); if (state.version === v) apply(data);`
- Two uncoordinated async functions mutating shared state interleave nondeterministically — serialize with a promise-chain mutex: `queue = queue.then(job)`.

## Timeouts & Cleanup

- `fetch` has no timeout: `fetch(url, {signal: AbortSignal.timeout(ms)})` rejects with TimeoutError AND aborts the request — unlike `Promise.race`, which abandons but never stops it (→ SKILL.md Traps).
- Combine cancellation causes with `AbortSignal.any([userSignal, AbortSignal.timeout(ms)])`.
- Runtime floors for AbortSignal helpers: `modern.md`.
