# Troubleshooting - Google Workspace CLI

Diagnose by error `reason` field, not HTTP status: Google reuses 403 for at least three unrelated failures.

## 403 Disambiguation

| `reason` | Actual problem | Fix |
|----------|----------------|-----|
| `accessNotConfigured` | API not enabled in the project | Open the `enable_url` from the error payload, enable the API, wait a few minutes for propagation, retry |
| `insufficientPermissions` / `ACCESS_TOKEN_SCOPE_INSUFFICIENT` | Token lacks the scope this method needs | Re-login with explicit `--scopes`; check the scope tier table in `auth-playbook.md` |
| `userRateLimitExceeded` / `rateLimitExceeded` | Quota, not permissions | Exponential backoff with jitter; add `--page-delay` on sweeps; do NOT change scopes |
| `domainPolicy` | Workspace admin policy blocks the API for this user | Escalate to the tenant admin — no client-side fix exists |

## invalid_grant on Login or Refresh

Symptom: previously working account fails; `gws auth status` shows expired/revoked.

- One-off: token revoked (password change, admin action) — `gws auth login --account <email>` again.
- Recurring every 7 days: the OAuth client is in Testing publishing status — see the OAuth client lifecycle section in `auth-playbook.md` for the real fix.

## 429 or Sustained Rate Limiting

- Retry only 429 and 5xx; back off exponentially with jitter and cap retries.
- For list sweeps, prefer fewer, larger pages (schema tells you the per-API max) plus `--page-delay` over many small pages.

## 404 on an Object You Can See in the Browser

- Drive: item lives on a shared drive — add `"supportsAllDrives": true` (and `"includeItemsFromAllDrives": true` on list calls).
- Any service: you resolved the id under a different account than the one executing — compare `gws auth list` default vs the `--account` used to find the id.

## 400 Invalid Params or Body

- Run `gws schema <service.resource.method>`; align `--params` and `--json` keys with schema; verify path params are present.
- Known required-param 400s: People API needs `personFields` on reads; Calendar `orderBy: "startTime"` needs `"singleEvents": true`.

## Empty Results That Should Not Be Empty

- Service account without domain-wide delegation lists its own empty Drive — wrong identity, not an error (`auth-playbook.md`).
- Drive v3 fields look missing → default field mask; pass explicit `fields`.
- `messages.list` "has no content" → it only returns id stubs by design; follow with `messages.get`.

## Wrong Account Used for Command

- Check `gws auth list`; set explicit default via `gws auth default <email>`; use `--account` for one-off override.

## Discovery Command Mismatch

Symptom: expected method not found.

- Verify service alias and API version against `command-index.md`.
- Discovery responses cache for 24 hours — a brand-new API method may not appear until the cache refreshes.
- Use `<api>:<version>` syntax for explicit version overrides.
