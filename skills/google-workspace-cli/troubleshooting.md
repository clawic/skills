# Troubleshooting — Error Chains

Diagnose by error `reason` field, not HTTP status: Google reuses 403 for at least three unrelated failures. The compact triage table lives in SKILL.md (Error Triage); this file is the deeper chains.

## 403 Disambiguation

| `reason` | Actual problem | Fix |
|----------|----------------|-----|
| `accessNotConfigured` | API not enabled in the project | Open the `enable_url` from the error payload, enable the API, wait a few minutes for propagation, retry |
| `insufficientPermissions` / `ACCESS_TOKEN_SCOPE_INSUFFICIENT` | Token lacks the scope this method needs | Re-login with explicit `--scopes`; check the scope tier table in `auth-playbook.md`; on Admin APIs, also check the ACCOUNT has admin privileges (`admin.md`) |
| `userRateLimitExceeded` / `rateLimitExceeded` | Quota, not permissions | Backoff formula in `quotas.md`; add `--page-delay` on sweeps; do NOT change scopes |
| `domainPolicy` | Workspace admin policy blocks the API for this user | Escalate to the tenant admin — no client-side fix exists |

## invalid_grant on Login or Refresh

Symptom: previously working account fails; `gws auth status` shows expired/revoked.

- One-off: token revoked (password change, admin action, security event) — `gws auth login --account <email>` again.
- Recurring every 7 days: the OAuth client is in Testing publishing status — the real fix is moving it to Production (`auth-playbook.md`), not re-logging in weekly. Fatal for automations (`automation.md` preflight).

## 429 or Sustained Rate Limiting

- Retry only 429 and 5xx: `sleep = min(2^attempt + rand(0..1s), 64s)`, cap at ~5 attempts (`quotas.md`).
- Recompute the sweep budget (`quotas.md`): fewer, larger pages up to the schema max, plus `--page-delay`.
- Parallel jobs against one user compound per-user quota — serialize per identity before adding delay.

## 404 on an Object You Can See in the Browser

- Drive: item lives on a shared drive — add `"supportsAllDrives": true` (and `"includeItemsFromAllDrives": true` on list). Full flag set in `drive.md`.
- Any service: you resolved the id under a different account than the one executing — compare `gws auth list` default vs the `--account` used to find the id.
- Drive: the id belongs to a shortcut whose target was deleted — dereference `shortcutDetails.targetId` (`drive.md`).
- Gmail: a stored `historyId` expired — full resync, store a new baseline (`gmail.md`).

## 400 Invalid Params or Body

- Run `gws schema <service.resource.method>`; align `--params` and `--json` keys; verify path params.
- Known required-param 400s: People reads need `personFields`; Calendar `orderBy: "startTime"` needs `"singleEvents": true`; Directory `users.list` needs `customer` or `domain` (`admin.md`); Sheets `values.update`/`append` need `valueInputOption` (`editors.md`).
- Gmail send 400s: `raw` encoded with standard base64 instead of base64url (`gmail.md`).

## 409 / 412 Precondition Failures

Stale read before write: the object changed (or Calendar's etag/sequence advanced) between your read and your write. Re-read, re-apply on fresh state (`calendar.md`, SKILL.md Rule 3). Do not retry blindly — the retry re-submits the stale write.

## Empty Results That Should Not Be Empty

- Service account without domain-wide delegation lists its own empty Drive — wrong identity, not an error (`auth-playbook.md`).
- Drive v3 fields look missing → default field mask; pass explicit `fields`. `nextPageToken` missing from a custom mask → pagination silently stops after page one (`drive.md`).
- `messages.list` "has no content" → id stubs by design; follow with `messages.get`.
- Gmail search finds nothing → the mail is in spam/trash, excluded by default: `includeSpamTrash` (`gmail.md`).
- Calendar instances missing → recurring series not expanded (`singleEvents`) or cancelled instances hidden (`showDeleted`) (`calendar.md`).
- Reports queries at incident time → data lags by application, minutes to days; re-run later before concluding "no activity" (`admin.md`).

## "It Succeeded but Nothing Happened"

- Event created, nobody invited → `sendUpdates` defaults to none (`calendar.md`).
- File shared, no email received → intended: `sendNotificationEmail: false`; or unintended mass mail: it defaults true (`drive.md`).
- Formula written to a cell shows as text → `RAW` instead of `USER_ENTERED` (`editors.md`).
- Sent mail threads as a new conversation in recipients' clients → missing `In-Reply-To`/`References` headers (`gmail.md`).

## Wrong Account Used for a Command

Check `gws auth list`; set the default via `gws auth default <email>`; `--account` for one-off override; `default_account` in config makes it structural.

## Discovery Command Mismatch

Expected method not found → alias table, 24-hour cache, `<api>:<version>` override (`command-index.md`).
