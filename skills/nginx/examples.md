# Configuration Traps

## Location Matching

- `location /api` matches `/api`, `/api/`, `/api-v2`, `/apiary` — broader than expected
- `location /api/` (with slash) does NOT match `/api` without a slash
- `location = /api` is exact but doesn't match `/api/`
- A regex `location ~` takes priority over a longer prefix — order matters

## Root vs Alias

- `root /var/www; location /img/` → looks in `/var/www/img/`
- `alias /var/www/; location /img/` → looks in `/var/www/` (replaces the location)
- `alias` without a trailing slash + location with a slash = malformed path
- `alias` with a regex requires a capture: `alias /var/www$1`

## try_files

- `try_files $uri /index.html` without `$uri/` — doesn't find directories
- The last argument is an internal redirect, not a file check — different behavior
- `try_files` + `proxy_pass` in the same location = try_files always wins
- `=404` as a fallback is a code, not a file — `/404` would be a file

## If Statement

- `if` creates a new context — inherited directives may not apply
- `if ($request_uri ~* \.php)` in an already-processed location = pointless double check
- `return` and `rewrite` inside `if` work — other directives are problematic
- Multiple `if` blocks don't combine with AND — each is independent

## Variables

- `$uri` is normalized (decoded) — `$request_uri` is raw
- Undefined variable = empty string, not an error
- `set` inside `if` always runs — only the block is conditional
- `map` is more efficient than multiple `if` for switch/case

## Includes

- `include /etc/nginx/conf.d/*.conf` — alphabetical order, may matter
- Include of a nonexistent file = nginx won't start
- A relative include is relative to nginx.conf, not the current file
- Circular includes = nginx won't start, with a confusing error
