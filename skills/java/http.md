# HTTP and Networking — Clients, Timeouts, DNS, TLS

Calling another service is where a JVM most often hangs. Composition of the calls is `async.md`; this is the transport underneath.

## Choosing a Client

| Client | Since | Use when |
|---|---|---|
| `java.net.http.HttpClient` | 11 | Default for new code: HTTP/2, sync and async, no dependency |
| Apache HttpClient / OkHttp | — | You need fine-grained connection-pool control, interceptors, or an ecosystem integration |
| Spring `RestClient` / `WebClient` | 6.1 / 5 | Inside Spring; `RestTemplate` is maintenance-only (`spring.md`) |
| `HttpURLConnection` | 1.1 | Never in new code — its timeout, redirect, and error-stream semantics are all traps |

```java
static final HttpClient CLIENT = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(3))          // connect only
        .followRedirects(Redirect.NORMAL)
        .build();

var req = HttpRequest.newBuilder(URI.create(url))
        .timeout(Duration.ofSeconds(10))                // the whole request: set it on EVERY request
        .header("Accept", "application/json")
        .GET().build();
```

- The client is immutable, thread-safe, and owns the connection pool: build ONE and share it. A client per call leaks connections and defeats keep-alive.
- `connectTimeout` on the builder and `timeout` on the request are different clocks; missing the second one means an infinite wait once the connection is established.

## Timeouts: the Four Clocks

1. **Connect** — TCP handshake. Seconds, not minutes.
2. **Request/read** — waiting for the response. This is the one people forget, and its default in most Java clients is unlimited.
3. **Pool acquisition** — waiting for a free connection from the client's pool. Under load this is the real latency.
4. **Caller budget** — the timeout of whoever called you, which must exceed the sum of the retries and backoffs below it (`async.md`).

- A missing read timeout turns one slow dependency into a thread-pool outage: every worker parks, and the service stops answering health checks even for endpoints that never touch the dependency.
- `HttpClient.sendAsync` with a request timeout still holds the connection until the timeout fires; a timeout is not a cancellation of the remote work.
- No response body read = no connection returned to the pool. Always consume or close the body, including on error responses (`io.md`).

## Status Codes and Error Bodies

- A 4xx/5xx is a normal response, not an exception: `HttpClient` returns it, and code that only checks for exceptions treats a 500 as success. Assert on `response.statusCode()` explicitly.
- `HttpURLConnection` reads error bodies from `getErrorStream()`, not `getInputStream()` — one more reason not to use it.
- Retry on 429/502/503/504 and connection failures; never on 400/401/403/422. Honour `Retry-After` when present (`async.md`).
- Decode the body only after checking the content type: an HTML error page parsed as JSON produces a misleading `JsonParseException` that hides the real 503 (`serialization.md`).

## DNS and Connection Reuse

- **The JVM caches DNS.** With a security manager historically installed, positive lookups were cached forever; the modern default is 30 seconds (`networkaddress.cache.ttl`), and failures are cached for 10 (`networkaddress.cache.negative.ttl`). A long-lived JVM behind a failover DNS name can keep hammering a dead IP: set `-Dnetworkaddress.cache.ttl=30` (or lower) explicitly rather than relying on the default of whatever JDK you are on.
- Keep-alive reuses connections; a connection idle longer than the server's or load balancer's idle timeout is closed remotely, and the next request fails once with a reset. Either set the client's idle eviction below the server's timeout, or retry idempotent requests on a connection reset.
- Connection pools are per-destination. A slow host does not have to starve the others — verify your client's per-route limits rather than assuming.
- Proxies come from `-Dhttp.proxyHost`/`https.proxyHost` plus `http.nonProxyHosts` (note: `https.nonProxyHosts` does not exist), or a `ProxySelector` on the client. In corporate networks a missing `nonProxyHosts` sends internal traffic through the proxy and fails in ways that look like DNS.
- IPv6-first resolution on a host with broken IPv6 gives long stalls followed by success; `-Djava.net.preferIPv4Stack=true` is the diagnostic switch.

## TLS

- Trust comes from the JDK's `cacerts`. An internal CA goes into a trust store (`-Djavax.net.ssl.trustStore=` + password) or the image's `cacerts` — never into a permissive `TrustManager` (`security.md`).
- `PKIX path building failed` = the chain is incomplete or the CA is not trusted: usually a server missing an intermediate certificate, which browsers fetch and Java does not.
- `unable to find valid certification path` after a JDK upgrade: the new JDK ships a different `cacerts`, or your custom store was in the old JDK's directory. Store it outside `$JAVA_HOME`.
- Handshake debugging: `-Djavax.net.debug=ssl:handshake` (verbose, non-production).
- Certificate expiry is a scheduled outage. Alert on it, and remember client certificates expire too.

## Servers and Inbound Traffic

- Thread-per-request servers are bounded by their pool: a slow downstream call inside a handler consumes a request thread. Virtual threads (21+) remove that coupling for blocking code (`concurrency.md`).
- Set request-body size limits, header limits, and idle timeouts at the server, not in application code — a client that connects and sends nothing costs a connection until something times it out.
- Backpressure: bound the inbound queue and reject with 503 rather than accumulating requests you cannot serve. An unbounded queue converts a latency spike into an OOM (`memory.md`).
- Graceful shutdown: stop accepting, finish in-flight requests with a deadline, then exit. Without it, every deploy drops the requests in flight.
- Health endpoints must check the dependencies that matter (pool, disk, downstream) but must not fan out expensive calls on every probe.

## Diagnosing a Slow or Failing Call

1. Reproduce with `curl -v` from the SAME host and user — it separates DNS, TLS, and application problems in one command.
2. Thread dump during the hang: `SocketRead` frames name the destination and prove where the wait is (`debug.md`).
3. Check pool metrics before blaming the network — waiting for a connection looks identical to a slow server from inside the application.
4. Enable client wire logging briefly, with redaction: headers carry the tokens you must not log (`logging.md`).
