# Performance Traps

## Worker Configuration

- `worker_processes auto` can be excessive in containers — limit it manually
- `worker_connections 1024` is per worker — total = workers × connections
- `multi_accept on` can cause starvation in some workers
- CPU pinning (`worker_cpu_affinity`) rarely helps in practice

## Buffers

- Small `client_body_buffer_size` + uploads = writes to tmp disk
- Silent buffer overflow — the request is processed but slower
- `proxy_buffer_size` must be large enough for response headers
- Many small buffers are worse than a few large ones for memory

## Gzip

- `gzip on` without `gzip_types` = only text/html compressed
- Gzip on already-compressed files (jpg, png, zip) = CPU waste + larger file
- `gzip_vary on` is critical for a CDN — without it, the CDN may serve the wrong version
- `gzip_comp_level 9` rarely worth it — 6 is the sweet spot

## Caching

- `proxy_cache` without `proxy_cache_key` uses a default that can collide
- A cache key with `$request_uri` includes the query string — can blow up the cache
- `proxy_cache_valid 200 1h` — but 301/302 are cached forever by default
- `inactive` vs `max_size` — inactive evicts by time, max_size by space

## Static Files

- `sendfile on` without `tcp_nopush on` = doesn't leverage the kernel optimization
- `expires` header without `Cache-Control` = browsers ignore it
- `try_files` with many paths = multiple syscalls
- Symlinks + `disable_symlinks` for security can break deploys

## Logging

- `access_log` is synchronous by default = I/O block on every request
- `buffer=32k flush=5s` for async logging — but you can lose logs on a crash
- Log rotation without a `reopen` signal = nginx writes to a deleted file
- JSON logging is parseable but more bytes per line
