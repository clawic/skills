# Auth Traps

## Choosing an OAuth Flow

| Caller | Flow |
|--------|------|
| Server-side web app | Authorization code |
| SPA or mobile app | Authorization code + PKCE (no client secret can live in the client) |
| Script/daemon, no user present | Client credentials |
| TV/CLI without a browser | Device code |
| Anything else / unsure | Authorization code + PKCE — it is never wrong, only sometimes unnecessary |

Implicit flow is legacy: it puts tokens in the URL fragment (leaks via history/referrer). If a provider still offers it, don't take it.

## Bearer Token

- `Authorization: Bearer:token` (with a colon) = WRONG, it's `Bearer token` (space)
- Token with a trailing newline (copy-paste) = mysterious 401; `echo -n` or trim first
- Bearer in query param `?token=x` works in some APIs but gets logged in access logs
- Token hardcoded in code gets committed to git; always an env var (naming: `credentials.md`)
- Basic auth is `base64(user:pass)` — the colon is part of the encoded string; encoding user and pass separately produces a valid-looking header that always 401s

## OAuth

- `state` parameter ignored = vulnerable to CSRF; generate per-request, validate on callback
- Token refresh without a mutex = race condition: N parallel requests each refresh, and providers that rotate refresh tokens invalidate all but the last winner — the others are logged out
- Refresh proactively when the token is within ~60s of `exp`, not reactively on 401: the 401-then-refresh-then-retry path doubles latency on every expiry and races under load
- Expired access token + expired refresh token = user must re-login (not just refresh)
- `offline_access` scope forgotten = no refresh token issued; you discover it an hour later

## JWT

- Verifying only the signature without validating `exp` = eternal tokens accepted
- `exp` is Unix seconds, not milliseconds: compare against `Date.now() / 1000`
- `aud` claim ignored = a token minted for another service is accepted by yours
- Algorithm confusion: token declares `alg: none` and a permissive library accepts it unsigned — pin the expected algorithm server-side, never read it from the token
- Decoding a JWT is not verifying it: `jwt.decode()` without the key check accepts anything

## API Keys

- API key in URL gets cached by proxies/CDNs, exposed in logs
- Rate limit per API key + key shared across clients = one noisy client starves the rest
- Key rotation without a grace period = downtime (procedure: `credentials.md`)
- API key without expiration + leak = permanent access until someone notices

## Session

- Predictable session ID = session hijacking
- Session not invalidated server-side on logout = stolen cookie stays valid
- Cookies without `Secure` + `HttpOnly` = interceptable over HTTP, readable by injected JS

## Headers

- `X-API-Key` vs `Api-Key` vs `apikey`: differs per API — check the service section, don't guess
- Auth header is not propagated on cross-origin redirects by most clients; a 302 to another host silently loses auth → follow the redirect target directly
- CORS preflight (OPTIONS) carries no auth headers; a backend requiring auth on OPTIONS produces a misleading "CORS error" whose real cause is a 401
