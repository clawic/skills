# Automation — Unattended Sweeps and Cron-Safe Scripts

A one-off command that worked is not an automation. Unattended runs add four failure modes interactive use never sees: expired auth with nobody to click, partial completion with no resume point, double-processing on rerun, and silent success that did nothing.

## Preflight (top of every scheduled script)

- `gws auth status` non-interactively; exit non-zero if the account isn't authenticated — a cron job must never trigger a browser OAuth flow.
- Explicit `--account` on every command (SKILL.md Rule 4) — cron inherits whatever default the last human left.
- Testing-status OAuth clients kill refresh tokens after 7 days (`auth-playbook.md`) — an automation on a Testing client is a weekly outage on a timer; move the client to Production before scheduling anything.

## Idempotency: Mark What You Processed

The loop is process → mark → verify mark, so a rerun after a crash skips completed work:

| Service | Marker | Skip filter |
|---------|--------|-------------|
| Gmail | user label via `batchModify` | `q: "-label:automation-processed"` |
| Drive | `appProperties` key | `q: "not appProperties has { key='processed' and value='true' }"` |
| Calendar | `extendedProperties.private` key | filter client-side on the property |
| Sheets | status column written with the row | re-read column before acting |

Mark AFTER the action succeeds, never before — a pre-marked failure is invisible forever; an unmarked success is a harmless duplicate check on rerun.

## Resumability

- Persist `nextPageToken` (or the last `modifiedTime` high-water mark) to a state file under `~/Clawic/data/google-workspace-cli/` after each page — a crash at page 40 of 50 resumes at 40, not 1.
- Steady-state jobs use the incremental APIs instead of re-listing: Drive `changes.list` with a stored page token, Gmail `history.list` with a stored `startHistoryId` (`drive.md`, `gmail.md`). Token expired (404) → full resync once, store a fresh baseline.

## Shell Discipline

```bash
set -euo pipefail
files=$(gws --account "$ACCT" drive files list \
  --params '{"q":"trashed = false","pageSize":100,"fields":"files(id,name),nextPageToken"}')
echo "$files" | jq -e '.files | length > 0' >/dev/null   # -e: empty result fails the run
echo "$files" | jq -r '.files[] | [.id,.name] | @tsv'     # TSV for downstream tools
```

- `jq -e` turns "empty when it shouldn't be" into a non-zero exit — cron surfaces it instead of mailing a blank report.
- Exit non-zero on ANY partial failure; count successes/failures and print both — "processed 96/100" in a log nobody reads is a silent incident.
- Quote every id; Drive ids are safe but names are not — never interpolate names into `q` without escaping single quotes.

## Pacing and Read-Modify-Write

- Apply the sweep budget formula from `quotas.md` before scheduling; set `--page-delay` and batch sizes from the math, not from the first 429.
- Re-read each target immediately before writing it (SKILL.md Rule 3) — objects change between the listing sweep and the action, especially in shared corpora; Calendar additionally enforces this via etag/sequence (`calendar.md`).

## Unattended Mail Is Drafts First

Scheduled jobs that compose mail create drafts; a human reviews in Gmail; sending is a separate, explicitly approved step (`gmail.md`). Fully unattended sending is reserved for workflows the user has named in config (`write_policy: open` plus an explicit no-go-zone check).

## Evidence Trail

- Append each run's command template, counts, and duration to `command-log.md`; deviations (count drift, new error signatures) go to `incidents.md` with the failing payload.
- Mutating automations log to `change-control.md` like any other write — the approval covers the recurring job, the evidence accrues per run.
