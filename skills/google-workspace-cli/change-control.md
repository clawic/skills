# Change Control — Gates and Evidence for Mutations

Applies to every command that sends, shares, updates, or deletes data. Reads need none of this (SKILL.md Rule 2 — approval theater on reads trains users to click through). The `write_policy` config variable selects how many gates run: `dry-run-first` (all), `confirm-only` (skip dry-run where preview adds nothing), `open` (log-only, user's explicit choice).

## Gate 1: Preflight

- Account and tenant confirmed — explicit `--account`, matching the task's tenant wall
- Method schema checked (`gws schema ...`); required fields present
- Targets are immutable ids, not display names (SKILL.md Rule 3); state re-read immediately before execution
- API enabled for the project (a 403 `accessNotConfigured` mid-mutation means preflight was skipped)

## Gate 2: Dry-Run / Preview

- `--dry-run` where the method supports it; record exact command and expected side effects
- Where `--dry-run` cannot preview the effect (mail delivery), use the domain's preview form: drafts for Gmail (`gmail.md`), a single-object trial for bulk sweeps
- Validate object count against expectation — a sweep that matched 4,000 objects when you expected 40 stops here

## Gate 3: Approval

- Restate affected resources and side effects; request explicit confirmation
- Block if any identifier is ambiguous or the account/tenant is unclear
- Notification side effects are part of the approval: `sendNotificationEmail` / `sendUpdates` behavior stated out loud (SKILL.md Traps)

## Gate 4: Apply and Verify

- Execute the approved command once; capture output
- Verify with at least one read-only command that the expected state change is visible
- Log the entry below; mismatch between expected and observed → `incidents.md`

## Log Entry Template

Record in `~/Clawic/data/google-workspace-cli/change-control.md`:

```markdown
## YYYY-MM-DD HH:MM - Operation
- command_id:
- account:
- tenant:
- mode: dry_run | apply
- command:
- impacted_objects:        # stable ids, never display names
- expected_side_effects:   # including notification emails
- dry_run_evidence:
- confirmation_token:      # copied exactly from user approval
- rollback_plan:
- result:                  # success/failure signal from command output
- follow_up_checks:
```

## Rollback Planning

One practical rollback path per mutation, chosen BEFORE apply:

| Mutation | Rollback |
|----------|----------|
| Trash (Drive/Gmail) | untrash — this is why trash beats delete (SKILL.md Rule 7) |
| Permanent delete | none — which is the argument for not doing it |
| Permission grant | `permissions.delete` with the recorded permissionId |
| Label/property change | inverse `batchModify` / property removal using the recorded id list |
| Sent mail / event invites | none for the send itself — communication rollback is a follow-up message; gate accordingly |
| User suspend | `"suspended": false`; user delete → `users.undelete` within 20 days (`admin.md`) |

## Recurring Automations

A scheduled mutating job passes the gates once for the job (design approval, recorded here), then accrues per-run evidence via its logs (`automation.md`). Any change to the job's scope, filter, or account re-triggers approval.
