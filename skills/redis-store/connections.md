# Connections — Pooling, Timeouts, Buffers, TLS

Most "Redis is down" incidents are connection incidents. The server is fine; something exhausted sockets, blocked the pool, or got disconnected by a buffer limit.

## Pooling

- Every connection costs a file descriptor on both sides and an output buffer on the server. Reuse them; opening one per request adds a TCP (and TLS) handshake to every call.
- Sizing: `pool_size ≈ concurrent in-flight commands per process`, which for a synchronous framework is the worker/thread count, not the request rate. A 16-worker process needs ~16-20 connections, not 200.
- Multiply by everything: `processes × pool_size × cluster_nodes` is what the server sees. 20 pods × 20 connections × 6 nodes = 2400 sockets.
- `maxclients` defaults to 10000 and is capped down to fit the process file-descriptor limit, with nothing but a startup-log warning — check `INFO clients` `maxclients` for the effective value.
- `ERR max number of clients reached` means the pool math above, a connection leak, or both. `CLIENT LIST` grouped by `addr` and `age` names the leaker; `name=` set by the client (`CLIENT SETNAME`) makes that diagnosis trivial and costs nothing.

## Connections That Must Not Be Shared

| Usage | Why it needs its own connection |
|---|---|
| `SUBSCRIBE`/`PSUBSCRIBE` (RESP2) | The connection enters subscriber mode and refuses normal commands |
| `BLPOP`, `BRPOP`, `BLMOVE`, `XREAD BLOCK` | It is parked on the server for the timeout; a pooled connection is unavailable for that whole time |
| `MULTI`/`WATCH` sequences | State lives on the connection; interleaving another command's `EXEC` breaks it |
| `MONITOR` | Streams every command; never in a shared pool, and never left running |

Blocking calls with a timeout of `0` block forever — the classic pool exhaustion. Always pass a finite timeout, shorter than the client's own socket timeout, or the client kills a connection the server still considers active.

## Timeouts

Set all four; a missing one is where a partial outage becomes a total one.

| Timeout | Where | Guidance |
|---|---|---|
| Connect timeout | Client | 1-2 s. Longer just makes a dead node look like a slow one |
| Command/socket timeout | Client | A few times your p99 command latency (typically 100-500 ms), and always longer than any `BLOCK`/`BRPOP` timeout you pass |
| `timeout` | Server | Idle client disconnect, default 0 (never). A non-zero value plus a pool that does not validate connections produces "connection reset" storms |
| `tcp-keepalive` | Server | Default 300 s; keeps NAT and load-balancer state alive and detects half-open sockets |

Retries: retry idempotent reads, never blindly retry writes. `INCR` retried after a timeout that actually succeeded double-counts — that is what idempotency keys are for.

## Client Output Buffers

The server buffers replies per client. Exceeding a limit **disconnects the client**, which the application sees as a mystery reset.

`client-output-buffer-limit <class> <hard> <soft> <soft-seconds>`, with defaults:

| Class | Default | Meaning |
|---|---|---|
| normal | `0 0 0` | Unlimited — a client requesting a huge reply can grow the server's memory until the OOM killer arrives |
| replica | `256mb 64mb 60` | A replica that cannot keep up is dropped and must full-resync |
| pubsub | `32mb 8mb 60` | The slow-subscriber disconnect |

Watch `client_recent_max_output_buffer` and `client_recent_max_input_buffer` in `INFO clients`. A rising replica buffer during writes means the backlog and the buffer need sizing together, not just the buffer.

The unlimited `normal` class is why one `LRANGE 0 -1` on a 2 GB list can take the server down with the memory the reply needs — the limit that would have protected you is off by default.

## TLS

- Adds a handshake per connection (another reason to pool) and per-message cost. Budget a measurable throughput reduction and re-benchmark rather than assuming a number.
- Client must trust the server's CA and, with `tls-auth-clients yes` (the default when TLS is on), present its own certificate.
- Certificate expiry takes down every client at once and looks exactly like an outage: monitor expiry dates as infrastructure, not as a security chore.
- `redis-cli --tls --cacert ca.crt --cert client.crt --key client.key` is the connectivity test that separates "TLS misconfigured" from "Redis down".
- Managed providers often offer in-transit encryption as a flag that also changes the port and endpoint.

## Diagnosing A Connection Problem

```bash
INFO clients          # connected_clients, blocked_clients, maxclients, buffer high-water marks
CLIENT LIST           # per connection: addr, name, age, idle, cmd, sub, multi, omem
CLIENT NO-EVICT on    # (>=7.0) protect an admin connection while you work
CLIENT KILL ID <id>   # surgical; CLIENT KILL LADDR/TYPE for a class
CLIENT UNPAUSE        # undo a CLIENT PAUSE left behind by a failover script
INFO stats            # rejected_connections, total_connections_received
```

- `blocked_clients` high and stable = blocking reads doing their job; high and *growing* = workers waiting on an empty queue while the pool starves everything else.
- `rejected_connections` rising is the `maxclients` ceiling, and it is the number to alert on — it precedes the outage.
- Connections with `age` in the thousands and `idle` also in the thousands are a leak; connections with a high `omem` are about to hit a buffer limit.
- Reproduce from the server host with `redis-cli` before touching the client library: if `redis-cli` connects and the app does not, the problem is TLS, auth, the pool, or DNS — not Redis.
