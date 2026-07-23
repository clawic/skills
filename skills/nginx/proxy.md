# Proxy Traps

## proxy_pass URL

- `proxy_pass http://backend` (no slash) — preserves `/api/users` → `/api/users`
- `proxy_pass http://backend/` (with slash) — replaces `/api/users` → `/users`
- Mixing `location /api/` with `proxy_pass http://x/v1` = unexpected paths
- Variables in proxy_pass (`$uri`) completely change the behavior

## Headers

- `proxy_set_header Host $host` — without this, the backend receives the proxy's IP as Host
- `Host $http_host` includes the port — `Host $host` doesn't
- `X-Forwarded-For` is overwritten, not appended — use `$proxy_add_x_forwarded_for`
- Headers with an underscore `_` are ignored by default — `underscores_in_headers on` to allow

## WebSocket

- Missing `proxy_http_version 1.1` = WebSocket fails silently
- `Connection "upgrade"` must be a literal string, not a variable
- Default 60s timeout kills idle WebSocket connections — raise `proxy_read_timeout`
- Multiple proxies in a chain = each one needs the upgrade headers

## Buffering

- `proxy_buffering on` (default) — full response before sending to the client
- With buffering, streaming responses don't work — SSE, chunked encoding broken
- `X-Accel-Buffering: no` header from the backend can disable it — but doesn't always work
- Buffer too small + large response = writes to temporary disk

## Timeouts

- `proxy_connect_timeout` default 60s — too long to detect a downed backend
- Slow backend + low `proxy_read_timeout` = frequent 504s
- A timeout in nginx doesn't cancel the request on the backend — it keeps processing

## Upstream

- Upstream server without a port = implicit port 80
- `max_fails=0` disables health checks — server never marked down
- `fail_timeout` is DOUBLE: the counting period AND the ban time
- Round-robin ignores weight if only 1 server is up
