# Resilience Traps

## Retry Logic

- Retry on POST/PUT without idempotency key = duplicates
- Immediate retry on 429 ignores `Retry-After` header = longer ban
- Retry on 400 Bad Request = waste, request is invalid
- Exponential backoff without jitter = thundering herd, everyone retries at the same time

## Timeouts

- Connect timeout too high = threads blocked waiting on DNS/TCP
- Read timeout includes server processing time, not just network
- No timeout = request hung forever if server doesn't respond
- Client-side timeout doesn't cancel the request on the server, it keeps processing

## Circuit Breaker

- Threshold too low = circuit opens on normal transient errors
- Half-open without a request limit = flood on the recovering server
- Circuit per host, not per endpoint = one bad endpoint affects all
- No circuit-state metrics = debugging impossible

## Rate Limiting

- Client-side rate limit without synchronization = exceed limit with multiple instances
- Local counter + distributed system = each node has its own counter
- Rate limit only on 429 response = you already exceeded the limit
- Backoff after 429 too short = extended ban

## Error Handling

- Generic catch that silences all errors = invisible bugs
- Retry that logs on every attempt = log flood during an outage
- Error in fallback handler = crash, not graceful degradation
- Async error without handler = unhandled rejection, process may die

## Connection Pooling

- Pool exhausted = requests queued or rejected
- Stale connection in pool = first request fails, next one OK
- Pool size too large = too many connections to the server
- No connection health check = dead connections in pool
