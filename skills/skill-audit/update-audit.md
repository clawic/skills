# Update Audit — Diffing a New Version Against What You Approved

A benign v1 says nothing about v2: updates are how compromised and sold accounts cash in on installed trust. The unit of audit is the diff; the unit of trust is still the whole artifact.

## Baseline

- Best: the audit log (`~/Clawic/data/skill-audit/audit-log.md`, written when `log_verdicts` is on) names the version you approved; your installed copy is that artifact.
- No log: the installed folder is the baseline.
- No installed copy either: it is not an update audit — run the full five passes as a new skill.

## Procedure

1. `diff -ru <installed>/ <new>/` — unified diff of the whole folder, hidden files included.
2. Added and modified lines: Passes 2-4 (language, execution, scope) on each hunk, read in file context — a hunk that looks clean alone can complete a payload split across an earlier, unchanged line.
3. New files: full audit, all five passes. New files are new skills.
4. Deleted lines: read them too. Removing a guardrail ("never run without confirming") changes behavior as surely as adding a payload.
5. Metadata diff: new declared paths, endpoints, bins, or env = new access, and it needs the same acceptance a fresh install would.
6. Honesty check: does the changelog describe this diff? A "typo fix" release that touches scripts or adds endpoints = flag on Rule 4 grounds, applied to release notes.

## Suspicion Multipliers (any one → full audit, not diff-only)

- A dormant skill (months without releases) suddenly updating
- Rapid-fire versions within days — probing what the scanner catches
- Publisher changed, or homepage/repo changed in metadata
- Diff size wildly exceeding the changelog's claim
- `strictness: paranoid` set (Configuration) — always full audit

## Verdict Semantics

- The verdict is on the UPDATE. Rejecting the update does not remove the installed version: the old artifact keeps its own verdict.
- Compromise evidence in the new version means the account is hostile now → treat the installed copy per `incident.md`: quarantine and re-audit rather than assume v-old innocence.
- Log a line either way (`report.md`): rejected updates are exactly the history that later proves a compromise pattern.
