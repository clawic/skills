# Errors — Code to Cause

Match on the numeric code or the code name; MongoDB rewords messages between versions. The dozen most common codes are in SKILL.md; this is the full working catalog plus the driver-side errors that never reach the server.

## Write Errors

| Code | Name | Cause and fix |
|---|---|---|
| 11000 | DuplicateKey | Unique index violated. Read `keyValue` in the error: `{field: null}` means two documents are missing the field, not that they collide — use a partial unique index (→ `indexes.md`). In an upsert race, retry once; the second attempt matches |
| 112 | WriteConflict | Two writers touched the same document under snapshot isolation. Outside a transaction the server retries internally; inside one, YOU retry the whole transaction (→ `transactions.md`) |
| 121 | DocumentValidationFailure | `$jsonSchema` rejected the write. `errInfo.details` names the failing path — the useful part; the top-level message never does |
| 66 | ImmutableField | An update tried to modify `_id`, or a shard key field outside a retryable write/transaction (→ `sharding.md`) |
| 40 | ConflictingUpdateOperators | Two operators in one update touch the same path (`{$set: {a: 1}, $inc: {"a.b": 1}}`). Split into two updates or one operator |
| 28 | PathNotViable | Dotted path crosses a non-object: `{$set: {"a.b": 1}}` where `a` is a string or an array of scalars |
| 2 | BadValue | Malformed argument — most often a projection mixing inclusion and exclusion, or an aggregation operator given the wrong arity |
| 16755 | Location16755 | Invalid GeoJSON geometry reaching a 2dsphere index (unclosed polygon, longitude before latitude) |

## Read and Cursor Errors

| Code | Name | Cause and fix |
|---|---|---|
| 50 | MaxTimeMSExpired | Your own `maxTimeMS` fired. The query is slow, not broken (→ `slow-queries.md`) |
| 43 | CursorNotFound | Cursor idle over 10 minutes, or the server that held it restarted. Page by range key instead of holding one cursor across slow work (→ `connections.md`) |
| 292 | QueryExceededMemoryLimitNoDiskUseAllowed | A blocking stage passed 100MB with disk use disabled (→ `aggregation.md`) |
| 96 | OperationFailed | Generic; on `find().sort()` it is usually the 32MB in-memory sort cap (→ `slow-queries.md`) |
| 262 | ExceededTimeLimit | A server-side operation (index build, `$lookup` sub-query, cursor `getMore`) exceeded its own budget |
| 133 | FailedToSatisfyReadPreference | No member matched the read preference, tag set, or `maxStalenessSeconds` (minimum 90 — a lower value is rejected, not clamped) |

## Topology and Failover Errors

| Code | Name | Cause and fix |
|---|---|---|
| 10107 | NotWritablePrimary | Write sent to a secondary or a node mid-stepdown. Almost always a connection string listing one host instead of the whole set with `replicaSet=` |
| 189 | PrimarySteppedDown | Failover in progress. Retryable writes cover single-document writes; `updateMany`/`deleteMany` do not retry (SKILL.md Consistency Model) |
| 91 | ShutdownInProgress | The node is going down, usually a rolling restart or maintenance — the driver should route elsewhere; if it does not, the URI is single-host again |
| 11602 | InterruptedDueToReplStateChange | The operation was in flight during an election; retry it |
| 6 / 89 | HostUnreachable / NetworkTimeout | Network, firewall, IP access list, or DNS — not the database (→ `connections.md`, `security.md`) |
| 13435 | NotPrimaryNoSecondaryOk | Read hit a secondary without a read preference allowing it |

## Auth Errors

| Code | Name | Cause and fix |
|---|---|---|
| 18 | AuthenticationFailed | Who you are: wrong password, wrong `authSource` (users live in the database that created them, usually `admin`), or a mechanism mismatch |
| 13 | Unauthorized | What you may do: authenticated but the role lacks the action. The message names the action and the namespace — grant that, not `root` (→ `security.md`) |
| 8000 | AtlasError | Atlas-layer rejection wrapping one of the above, plus its own: IP not in the access list, tier limit reached, user without database access (→ `atlas.md`) |

## Transaction Errors

| Code | Name | Cause and fix |
|---|---|---|
| 251 | NoSuchTransaction | The transaction expired past `transactionLifetimeLimitSeconds` (60s default) or the session ended. Do not retry the commit — retry the whole transaction |
| 24 | LockTimeout | The transaction waited too long for a lock another writer holds (default 5ms of patience inside a transaction, by design) |
| — | TransientTransactionError (label) | Retry the whole transaction. Match the LABEL, never a code list (→ `transactions.md`) |
| — | UnknownTransactionCommitResult (label) | Retry the COMMIT only; commits are idempotent by transaction number |

## Driver-Side Errors (the server never saw these)

- **`MongoServerSelectionError` / `ServerSelectionTimeoutError`** — the driver could not find a suitable server within `serverSelectionTimeoutMS` (30s default). The `reason` field is the real message: DNS failure, TLS handshake, auth on the handshake, or every node reporting itself as unknown. On Atlas, the top three causes are an IP not in the access list, a `mongodb+srv` lookup blocked by the network, and a stale TLS/CA chain.
- **`MongoNetworkTimeoutError` / socket timeouts** — an operation exceeded `socketTimeoutMS`. Setting this below your slowest legitimate query turns healthy work into random failures; bound the query with `maxTimeMS` instead (→ `connections.md`).
- **`MongoParseError`** — malformed URI. Passwords with `@`, `:`, `/`, or `%` must be percent-encoded.
- **`PoolClearedError` / "connection pool was cleared"** — the driver reset the pool after a topology change; it is a symptom of the failover or network blip that preceded it.
- **Mongoose "Operation buffering timed out after 10000ms"** — the model was used before the connection was established. Mongoose buffers rather than failing fast; the timeout is the buffer, not the database (→ `connections.md`).

## Reading Any Unknown Error

1. Extract `code` and `codeName` from the raw error object — ORMs and frameworks routinely reformat the message and drop both.
2. Bulk operations report per-document results: `writeErrors[]` carries an `index` into your input batch. One duplicate key in a 1,000-document `insertMany` fails at that index and, in ordered mode, stops there (→ `migrations.md`).
3. Check whether it carries an error LABEL (`TransientTransactionError`, `RetryableWriteError`) — labels are the retry contract; codes are the diagnosis.
4. Reproduce against a scratch database with the same server version before changing production; several of these codes change behavior across versions (→ `replication.md`, upgrade section).
