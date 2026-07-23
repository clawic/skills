# Auth Traps

## Bearer Token

- `Authorization: Bearer:token` (with a colon) = WRONG, it's `Bearer token` (space)
- Token with a trailing newline (copy-paste) = mysterious 401
- Bearer in query param `?token=x` works in some APIs but gets logged in access logs
- Token hardcoded in code gets committed to git, always use an env var

## OAuth

- `state` parameter ignored = vulnerable to CSRF, always validate
- Token refresh without a mutex = race condition, multiple simultaneous refreshes
- Expired access token + expired refresh token = user must re-login (not just refresh)
- `offline_access` scope forgotten = no refresh token

## JWT

- Verifying only the signature without validating `exp` = eternal tokens accepted
- `exp` in seconds, not milliseconds, `Date.now()` / 1000
- `aud` claim ignored = token for another service accepted
- Algorithm confusion: token says `alg: none` and the server accepts it unsigned

## API Keys

- API key in URL gets cached by proxies/CDNs, exposed in logs
- Rate limit per API key + key shared across clients = shared limit
- Key rotation without a grace period = downtime
- API key without expiration + leak = permanent access

## Session

- Predictable session ID = session hijacking
- Session not invalidated on logout = reusable
- Session timeout too long + shared computer = risk
- Cookies without the `Secure` flag sent over HTTP = interceptable

## Headers

- `X-API-Key` vs `Api-Key` vs `apikey`: differs per API, case-sensitive
- Auth header not propagated to redirects by default, a 302 loses auth
- CORS preflight doesn't include auth headers, confusing CORS error if the backend expects auth on OPTIONS
