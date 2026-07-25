# Connections — Drivers, Pools, Timeouts, and ODMs

More production MongoDB incidents come from how the application connects than from anything the database does. The whole file is one theme: the driver has opinions and defaults, and the defaults were chosen for a single process.

## The Connection String

```
mongodb+srv://user:pass@cluster.example.net/?retryWrites=true&w=majority&appName=orders-api&maxPoolSize=20
```

- Always the FULL replica set: `mongodb+srv://` (DNS resolves the members) or `mongodb://host1,host2,host3/?replicaSet=rs0`. A single-host URI cannot fail over, and the symptom arrives later as error 10107 (→ `errors.md`).
- `appName` on every client. It is how you identify the offender in `db.currentOp()` during a connection storm (→ `incidents.md`), and it costs one parameter.
- `w=majority&retryWrites=true` explicitly (SKILL.md rule 4) — never inherited from server defaults that change across versions.
- Percent-encode the password. `@`, `:`, `/`, `?`, `#`, `%` in a raw password produce a `MongoParseError` that reads like a syntax bug.
- `mongodb+srv` needs outbound DNS SRV and TXT lookups. Networks that block them fail with server selection timeouts and no useful message — test with a plain `mongodb://` URI listing hosts to isolate it.

## Pool Sizing

Formula: **server connections = processes × `maxPoolSize`** (driver default 100 per client per node).

- 40 pods × 100 = 4,000 connections, roughly 4GB of mongod RAM before serving a document (SKILL.md rule 8).
- Size from concurrency, not from traffic: a process handling 20 concurrent database operations needs `maxPoolSize: 20`, however many requests per second it serves. Connections are not throughput; they are simultaneity.
- `minPoolSize` above 0 keeps warm connections and removes the first-request TLS handshake — worth it for latency-sensitive services, wasteful for a fleet of mostly-idle workers.
- `maxIdleTimeMS` retires idle connections so a traffic spike does not leave the pool permanently inflated.
- When the pool is exhausted, operations queue and then fail with a pool timeout. That error means "your app is more concurrent than its pool", not "the database is slow".

## Timeouts, and Which One Actually Fires

| Setting | Default | What it bounds |
|---|---|---|
| `serverSelectionTimeoutMS` | 30000 | Finding a suitable server. Firing here means topology, DNS, TLS, or access list — not your query |
| `connectTimeoutMS` | 10000 | The TCP + TLS handshake to one node |
| `socketTimeoutMS` | none in most drivers | Inactivity on an established socket. Setting it below your slowest legitimate operation turns healthy work into random failures |
| `maxTimeMS` (per operation) | none | The SERVER cancels the operation. This is the correct way to bound a query |
| `waitQueueTimeoutMS` | varies | How long an operation waits for a free pooled connection |
| `heartbeatFrequencyMS` | 10000 | How often the driver re-checks each node's state |

- Bound queries with `maxTimeMS`, not with `socketTimeoutMS`. The first cancels work on the server; the second abandons a socket while the server keeps burning CPU on a query nobody will read.
- Newer drivers offer a single `timeoutMS` covering the whole operation including retries. Prefer it where available — the per-knob model above is where people accidentally create a timeout budget that exceeds the client's own request deadline.
- Rule of thumb: client HTTP deadline > `maxTimeMS` > expected query time. When the outermost deadline is the smallest, you get abandoned queries that still consume the database.

## Cursors

- A cursor idles out after 10 minutes and then returns error 43 on the next `getMore` (→ `errors.md`). Iterating a large result while doing slow per-document work is the classic way to hit it.
- The fix is not `noCursorTimeout` — that leaks server-side cursors when the client dies. The fix is range pagination: one query per page, keyed on the sort field (SKILL.md Query Semantics).
- First batch is small (about 101 documents or 1MB); later batches are bigger. `batchSize` tunes round trips versus memory; the default is right until profiling says otherwise.
- Always close cursors explicitly in long-lived processes, or use the language's iteration construct that closes on exit.

## Serverless and Short-Lived Processes

- Cache the client OUTSIDE the handler, in module scope. A client created per invocation opens a fresh pool, does a fresh handshake, and never reuses anything — the connection cap arrives long before the CPU limit does.
- Set `maxPoolSize` low (1-5) for serverless: each concurrent invocation is its own process, so the fleet multiplies the pool by the concurrency limit.
- Do not `await client.close()` at the end of a handler. Let the runtime freeze the container with the pool intact.
- Cold-start latency is dominated by TLS and SRV resolution, not by MongoDB. `minPoolSize: 1` on a warm container removes it from the second request onward.
- Atlas's connection limits are per cluster tier and are the real ceiling for large serverless fleets (→ `atlas.md`).

## ODM Traps

**Mongoose**

- "Operation buffering timed out after 10000ms" means the model was used before connecting. Mongoose BUFFERS operations instead of failing fast; the timeout is the buffer, not the database. Set `bufferCommands: false` in production so the real connection error surfaces.
- `autoIndex` defaults to true: every process boot issues index builds for every schema. Harmless on a laptop, a thundering herd on a 40-pod deploy. Turn it off in production and manage indexes as migrations (→ `migrations.md`).
- `lean()` returns plain objects and skips hydration — often several times faster for read-only paths, and the reason a "slow query" is sometimes not a query at all.
- Middleware (`pre`/`post` hooks) does not run for `updateMany`, `bulkWrite`, or raw driver calls. Validation and timestamps you believe are enforced are enforced only on the paths that go through documents.
- Mongoose casts values to the schema type, which incidentally blocks the `{$ne: null}` injection shape. That protection disappears the moment you drop to the raw driver (→ `security.md`).

**Prisma, Beanie, Spring Data and friends**

- Every ODM generates queries you did not write. Read the emitted query at least once per hot path; ODM-generated `$lookup` chains and `replaceOne`-instead-of-`updateOne` are common and invisible.
- Relation traversal in an ODM is N round trips unless it maps to `$lookup` or an embedded field. The document store amplifies the N+1 problem rather than hiding it.
- Schema-first ODMs will happily model a relational design in MongoDB. The ODM is not the place to make the embed-vs-reference decision (→ `schema.md`).

## Read Preference and Tags

- `primary` (default) · `primaryPreferred` · `secondary` · `secondaryPreferred` · `nearest`.
- `maxStalenessSeconds` bounds how far behind a chosen secondary may be; the minimum accepted value is 90, and a lower one is rejected rather than clamped.
- Tag sets route reads to specific members (analytics node, local region). A tag set matching no available member produces error 133, not a fallback — spell the fallback explicitly.
- Read preference is per operation, not just per client: set `primary` on the read-after-write path even in a service that reads secondaries everywhere else (SKILL.md rule 7).

## Verifying a Connection Setup

- `db.serverStatus().connections` on the server, `client.topology` or the driver's connection-pool events on the client — check both sides agree.
- Count connections per `appName` (→ `mongosh.md`) and compare against processes × `maxPoolSize`. A mismatch means clients are being created somewhere you did not expect.
- Test a failover deliberately in staging: `rs.stepDown()` on the primary and confirm the application recovers without restarting. Almost every single-host URI in production was discovered this way, or during the outage.
