# SSL Traps

## Certificates

- `ssl_certificate` must be the fullchain (cert + intermediate) — cert only = warning in browsers
- Chain order matters: cert first, then intermediates, NOT root
- Key and cert mismatch = nginx won't start, with a cryptic error
- Expired cert = nginx starts OK but browsers reject it

## Configuration

- `ssl on` is deprecated — use `listen 443 ssl` instead
- `ssl_protocols TLSv1 TLSv1.1` = insecure — minimum TLSv1.2
- `ssl_prefer_server_ciphers on` no longer recommended for TLS 1.3 — only for 1.2
- Misordered `ssl_ciphers` = the server picks a weak cipher

## OCSP Stapling

- `ssl_stapling on` without `ssl_trusted_certificate` = stapling silently disabled
- A DNS resolver is needed for OCSP — `resolver 8.8.8.8` if there's no local one
- Some CAs' OCSP is slow/unreliable — can add latency

## HTTP/2

- `http2` is a directive on listen, not a separate module
- `http2_push` deprecated in nginx 1.25+
- HTTP/2 + proxy to an HTTP/1.1 backend works — but loses multiplexing
- `large_client_header_buffers` may need tuning for HTTP/2

## Client Certificates

- `ssl_verify_client on` rejects without a cert — use `optional` to make it optional
- `ssl_client_certificate` is the CA, not the client's cert
- `$ssl_client_verify` is "SUCCESS", not "true" or "1"
- CRL checking requires extra config — without it, revoked certs are accepted

## Common Mistakes

- Redirect loop: 80→443→80 because `X-Forwarded-Proto` isn't checked
- Mixed content: an HTTPS page loads HTTP resources — `upgrade-insecure-requests` helps
- HSTS with a very long max-age + invalid certificate = inaccessible site
- Cert for `example.com` doesn't cover `www.example.com` — needs a SAN or wildcard
