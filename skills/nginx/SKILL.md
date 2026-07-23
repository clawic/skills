---
name: nginx
slug: nginx
version: 1.0.3
description: >-
  Configures Nginx for reverse proxy, load balancing, SSL/TLS termination, caching, and static
  serving. Use when writing or debugging nginx.conf, server blocks, proxy_pass, WebSocket
  proxying, 502/504 errors, or performance tuning.
homepage: https://clawic.com/skills/nginx
changelog: Deeper config patterns and performance guidance
metadata:
  clawdbot:
    emoji: 🌐
    displayName: Nginx
---

## When To Use

- Writing or reviewing nginx config: server blocks, locations, proxy_pass, upstreams
- Debugging 502/504/413/redirect loops behind an nginx reverse proxy
- SSL/TLS termination, HTTP/2, WebSocket, or SSE through nginx
- Tuning: workers, buffers, gzip, proxy cache, rate limiting
- Not for certificate issuance/renewal itself (ACME, Let's Encrypt) — nginx consumes certs; issuance is the `ssl` skill

## Quick Reference

| Situation | Play |
|-----------|------|
| Random 502 after a container/backend redeploy | DNS cached at startup — use variable in `proxy_pass` + `resolver` (`proxy.md`) |
| 502 immediately vs 504 after ~60s | 502 = refused/reset/bad response; 504 = timeout. Different fixes (`proxy.md`) |
| 413 on uploads | `client_max_body_size` — default is 1m; raise in `http` or the exact `server`/`location` |
| WebSocket connects then dies, or never upgrades | Upgrade trio + timeout (`proxy.md`) |
| SSE/streaming arrives all at once | `proxy_buffering off` for that location (`proxy.md`) |
| Wrong file served / 403 on aliased path | `root` vs `alias` semantics and the alias-traversal slash bug (`examples.md`) |
| Request hits wrong location block | Re-derive with the matching algorithm below; `nginx -T` to see effective config |
| Backend receives wrong path (`/api/api/...` or missing prefix) | proxy_pass trailing-slash rules (below) |
| Browser cert warning, curl works | Missing intermediates — serve fullchain (`ssl.md`) |
| 80→443 redirect loop behind a CDN/LB | Trust `X-Forwarded-Proto`, don't redirect on `$scheme` alone (`ssl.md`) |
| Slow under load, high CPU or connection errors | `performance.md` |
| Security headers vanished on some routes | `add_header` inheritance trap (`examples.md`) |
| Config behaves unlike it reads (`if`, variables, includes, root/alias) | `examples.md` |
| Anything else | Debugging Order below, then the closest file above |

## Core Rules

1. `nginx -t && nginx -s reload` — never restart to apply config; reload is graceful (old workers finish in-flight requests). Test first: a bad config on restart takes the site down; on reload it's rejected.
2. Read the effective config with `nginx -T`, not the files — includes, inheritance, and distro defaults (`/etc/nginx/conf.d/*`) mean the file you're editing may not be what runs.
3. One canonical `Host` line: `proxy_set_header Host $host;`. Without it the backend sees the upstream name from `proxy_pass` — breaks virtual hosts, redirects, and anything that reads Host.
4. `proxy_set_header` in a location wipes ALL inherited proxy headers from server/http level, same for `add_header`. Inheritance is all-or-nothing per level: if you set one header in a location, re-declare the full set there.
5. Sizing: max concurrent proxied clients ≈ `worker_processes × worker_connections / 2` (each proxied request holds a client fd and an upstream fd). 4 workers × 1024 connections → ~2048 clients. Set `worker_rlimit_nofile` ≥ 2× worker_connections.
6. Never put logic in `if` beyond `return`/`rewrite` — `if` in location context creates a pseudo-location where other directives misbehave. Use `map` for conditionals.
7. Diagnose from the log split: log `$request_time` and `$upstream_response_time` together. High request_time + low upstream_time = slow client or buffering problem; both high = slow backend. Without both numbers you're guessing which side is slow.

## Location Matching (the real algorithm)

Common misreading is "regex beats prefix". Actual order:

1. Exact `= /path` — match ends immediately.
2. Find the LONGEST matching prefix (order in file irrelevant for prefixes).
3. If that prefix is marked `^~` — use it, skip regex entirely.
4. Otherwise try regex locations `~` / `~*` in FILE ORDER — first regex match wins.
5. No regex matched — fall back to the longest prefix from step 2.

Consequences:
- `location /api` also matches `/api-v2`, `/apiary` — prefix is string prefix, not path segment. Use `location /api/` plus `location = /api` if you need the segment.
- A short regex declared early beats your long careful prefix — `^~` on static asset prefixes is the standard defense.
- `location /api/` does not match `/api` (no trailing slash) — pair with exact match or accept the 404.

## proxy_pass Path Rules

- `proxy_pass http://backend;` (no URI part) → request path passed unchanged: `/api/users` → `/api/users`.
- `proxy_pass http://backend/;` (any URI part, even just `/`) → matched location prefix is REPLACED by that URI: `location /api/` + `.../` → `/api/users` becomes `/users`.
- URI part inside a regex location or inside `if` = config error at startup — rewrite instead, or drop the URI part.
- With a variable in `proxy_pass` (`set $up http://backend; proxy_pass $up;`) path handling changes again: nginx passes the URI as given in the directive; combine with `$request_uri` explicitly if needed.
- Verify with the backend's access log or `curl -v` against the backend directly — not by reasoning about the config.

## Proxy Headers & Real IP

Canonical block (re-declare wholesale wherever any `proxy_set_header` appears — rule 4):

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

- `$host` = lowercase, no port; `$http_host` = raw header with port. APIs that generate absolute URLs usually need `$http_host`.
- `X-Forwarded-For` is client-spoofable. If nginx is behind a trusted LB/CDN, use the realip module: `set_real_ip_from <LB subnet>; real_ip_header X-Forwarded-For; real_ip_recursive on;` — otherwise rate limits and logs key on the LB's IP.
- Headers with underscores are silently dropped by default (`underscores_in_headers on` to keep) — a classic "auth works with curl -H, fails through nginx" cause.

## Upstream & Keepalive

```nginx
upstream backend {
    server 10.0.0.2:3000 max_fails=3 fail_timeout=10s;
    server 10.0.0.3:3000 max_fails=3 fail_timeout=10s;
    keepalive 32;
}
```

- `keepalive 32` does NOTHING alone. The trio: `keepalive N` in upstream + `proxy_http_version 1.1;` + `proxy_set_header Connection "";` in the location. Missing either latter directive = a new TCP (and TLS) handshake per request, silently.
- `keepalive N` = idle connections kept per worker, not a connection limit.
- Defaults: `max_fails=1 fail_timeout=10s`. `fail_timeout` is dual-purpose: the window for counting failures AND the ban duration. `max_fails=0` disables marking down entirely.
- These are passive checks (real requests fail first). Active health checks are nginx Plus only — in OSS, put a real health endpoint behind your monitoring instead.
- Server without a port = port 80 — a common surprise when the app listens on 3000.
- Retries: on error/timeout nginx tries the next upstream server. Non-idempotent methods (POST, PATCH, LOCK) are not retried since nginx >=1.9.13 unless you set `proxy_next_upstream non_idempotent` — do not set it for endpoints with side effects.

## try_files & Static

- SPA: `try_files $uri $uri/ /index.html;` — file, then directory (needs `index`), then internal fallback. Last arg is a redirect/code, not a checked file: `=404` to error instead.
- `try_files` + `proxy_pass` in one location: try_files controls; route to the proxy via a named location — `try_files $uri @app;` + `location @app { proxy_pass ...; }`. This is the canonical "static if present, else app" pattern.
- Static asset locations: `^~` prefix, `access_log off;`, `expires 30d;` + `add_header Cache-Control "public, immutable";` for hashed filenames (expires alone without Cache-Control gets ignored by some clients).
- `sendfile on; tcp_nopush on;` together — sendfile without tcp_nopush leaves the kernel optimization on the table.

## SSL/TLS Essentials

- `ssl_certificate` takes the FULLCHAIN (leaf + intermediates, leaf first, no root). Leaf-only "works" in browsers with cached intermediates and fails on fresh clients — the classic "works for me, warning for users".
- Baseline: `ssl_protocols TLSv1.2 TLSv1.3;` and start from the Mozilla SSL config generator (intermediate profile) rather than hand-picking ciphers.
- `ssl_ciphers` only governs ≤TLS1.2; TLS1.3 suites need `ssl_conf_command Ciphersuites` (OpenSSL) — a source of "my cipher config does nothing".
- `ssl_prefer_server_ciphers off` is the modern recommendation (client-hardware-aware selection); `on` was TLS1.2-era advice.
- `ssl_session_cache shared:SSL:10m;` — ~4000 sessions per MB, shared across workers. Skipping it costs a full handshake per returning client.
- Redirect loops, HSTS rollout, OCSP, client certs → `ssl.md`.

## Rate Limiting

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
location /api/ { limit_req zone=api burst=20 nodelay; limit_req_status 429; }
```

- `rate=10r/s` is enforced per-millisecond: 1 request per 100ms. Two instant requests without `burst` = second one rejected. Always pair rate with a burst sized for legitimate client behavior (a browser page load fires 10-30 parallel requests).
- `nodelay` = serve the burst immediately, refill over time; without it, burst requests queue and add latency.
- Default rejection status is 503 — set 429 so clients and monitoring can tell throttling from outage.
- Behind a CDN/LB without realip configured, `$binary_remote_addr` is the LB's address — you rate-limit everyone as one client.

## Debugging Order

1. `nginx -t` — config valid? `nginx -T` — what's actually loaded?
2. `tail -f` the error log; raise level only on the suspect vhost. Grep-to-cause:

| Error log message | Meaning |
|---|---|
| `connect() failed (111: Connection refused)` | Backend down or wrong port → 502 |
| `no live upstreams` | All upstream servers marked failed (max_fails tripped) → 502 |
| `upstream prematurely closed connection` | Backend crashed mid-response or app timeout shorter than nginx's → 502 |
| `upstream timed out (110)` | nginx waited `proxy_read_timeout` (default 60s) → 504 |
| `client intended to send too large body` | `client_max_body_size` (default 1m) → 413 |
| `worker_connections are not enough` | Connection budget exhausted → see Core Rule 5 |
| Anything else | Reproduce with `curl -v` against nginx AND directly against the backend; the diff localizes the fault |

3. A timeout in nginx does not cancel the backend request — the backend keeps burning CPU on a request nobody will read. Fix the slow endpoint, don't just raise the timeout.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `worker_processes auto` in containers | Reads host CPU count, not cgroup quota — 64 workers on a 2-CPU limit | Set explicitly to the container's CPU limit |
| Editing config, reloading, no change | Distro includes another file that wins | `nginx -T \| less`, find who owns the directive |
| Hostname in `proxy_pass` to dynamic infra | Resolved once at startup, cached forever | Variable + `resolver` (`proxy.md`) |
| `gzip on` for images/zip/woff2 | Recompressing compressed data: CPU spent, bytes gained | `gzip_types` with text formats only |
| `if` for routing logic | Pseudo-location; directives inside behave unpredictably | `map` + variable, or separate locations |
| Log rotation without signal | nginx keeps writing to the deleted inode; disk fills with no visible file | `nginx -s reopen` (USR1) in the rotate script |
| Reload "not taking" with WebSockets | Old workers stay alive until long-lived connections close | `worker_shutdown_timeout 30s;` to bound the drain |
| No `default_server` defined | First server block silently catches all unmatched Hosts | Explicit `listen 80 default_server; return 444;` catch-all |

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):

- `ssl` — certificate issuance, renewal, and TLS debugging beyond nginx directives
- `docker` — nginx in containers: images, networks, and the resolver at 127.0.0.11
- `caddy` — when automatic HTTPS and a simpler config beat nginx's control
- `vps` — server provisioning and hardening around the nginx install
- `dns` — records and propagation issues upstream of the proxy

---

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/nginx.
