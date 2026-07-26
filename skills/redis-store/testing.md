# Testing — Isolation, Determinism, Failure Injection

Two decisions cover most of it: run a real Redis or a fake one, and isolate tests by key namespace or by instance.

## Real Server Or Fake

| Option | Fidelity | Use when |
|---|---|---|
| Real Redis in a container (Testcontainers or a compose service) | Full: encodings, expiry, Lua, errors | Default. Startup is a second or two, amortized across the suite |
| Real Redis, one process shared by the suite | Full, but shared state | Fast local loops, with prefix isolation below |
| In-memory fake (fakeredis and friends) | Partial: command coverage lags, Lua and Cluster semantics differ | Unit tests of code paths where Redis is incidental, never for behaviour you actually depend on |
| Mocked client | None — you are testing your mock | Only to assert that a call was made, not what it does |

The rule that saves debugging time: any behaviour you rely on (TTL semantics, atomicity, `WRONGTYPE`, eviction, script errors) gets tested against a real server. A fake that returns `OK` for a command it does not implement is worse than no test.

## Isolation

- **Prefix per test**: every key the test writes starts with `test:<uuid>:`. Teardown is a `SCAN`+`UNLINK` over that prefix. Works with parallel tests on one server, works in Cluster.
- **Numbered database per worker**: `SELECT <worker_id>` is convenient and creates a habit that breaks under Cluster. Acceptable for a local suite, not for code that reaches production.
- **Instance per worker**: one container per parallel worker. Most isolation, most resources, and the only option that lets a test call `FLUSHDB`.
- **Never `FLUSHALL` in a suite that shares a server** — it deletes other workers' data and turns test failures into a race you will spend a day on.

Teardown must be unconditional (a fixture, not the last lines of the test body) or a failing test poisons the next one.

## Determinism Around Time

- Do not `sleep(2)` to test a 1-second TTL. Assert on the TTL instead: `PTTL key` within an expected range, or set the TTL to something the test controls.
- Expiry is lazy plus a sampling cycle, so "the key should be gone by now" is a flaky assertion. Assert on the read behaving as a miss, which is guaranteed.
- Self-hosted only: `DEBUG SET-ACTIVE-EXPIRE 0` disables the active cycle so you can test lazy-expiry behaviour deliberately. `DEBUG SLEEP <s>` blocks the server to test client timeouts. `DEBUG OBJECT <key>` exposes encoding internals. All three are unavailable on managed platforms.
- Pass timestamps into Lua as arguments rather than calling `TIME` inside, so tests can drive the clock.

## What To Actually Test

- **TTL is set where you think it is.** One assertion per cache write path: `PTTL` is positive. This single test catches the most common production leak (Core Rule 1).
- **Atomicity under concurrency.** Fire N parallel workers at the counter, lock, or claim, and assert the invariant (exactly one winner, total equals N). A lock that only works serially passes every serial test.
- **The eviction case.** A test instance with `maxmemory 4mb` and `allkeys-lru` proves your lock, queue or session code notices a missing key instead of crashing.
- **Reconnection.** Kill the container (or `CLIENT KILL` your own connection) mid-suite and assert the client recovers. Most incident postmortems end here.
- **Error paths.** Force `WRONGTYPE` by writing the wrong shape, `NOSCRIPT` by `SCRIPT FLUSH`, `READONLY` by pointing at a replica. Handling these is code, so it deserves tests.
- **Serialization round trips.** Whatever your client does with bytes, unicode and integers — assert it, once, so a library upgrade cannot change it without a failing test.

## Failure Injection

```bash
redis-cli DEBUG SLEEP 5           # server blocked: does your client time out and retry?
redis-cli CLIENT PAUSE 3000       # all clients paused: failover-like stall without killing anything
redis-cli CLIENT KILL TYPE normal # every app connection dropped: does the pool recover?
redis-cli CONFIG SET maxmemory 1mb    # instant OOM conditions on a scratch instance
docker stop <redis> && docker start <redis>   # cold start: does the app survive an empty cache?
```

The cold-start test is the one teams skip and then discover during an incident: an empty cache means every request hits the source of truth at once (stampede).

## CI

- Service container in the pipeline is enough; a cluster is not needed unless you rely on cluster semantics — in which case a 3-master `redis-cli --cluster create` in CI is worth the minute it costs.
- Pin the image tag to the version you run in production, and bump it deliberately: an implicit `latest` turns an upstream default change into a mystery failure.
- Health-gate the suite on `redis-cli PING` returning PONG rather than a fixed sleep.
- Keep test data small. A suite that loads a million keys per run is measuring your CI runner, not your code.

## Load Testing

- `redis-benchmark` measures Redis, not your application: report the pipeline depth and payload size with every number, or the result is meaningless.
- To measure your application's use of Redis, replay a realistic command mix against a dataset of realistic size — an empty instance has no eviction, no fragmentation and a warm allocator.
- Test with the same client library, pool size and TLS setting as production; each of the three moves the number.
