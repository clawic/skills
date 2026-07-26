# Between Runs — Recurrence, Acceptance, Cadence, And The Report A Human Reads

One audit is a snapshot; the value is in the second one. This file is what makes run N+1 cheaper and sharper than run N.

**Before closing any run**, read `## Open Findings`, `## Accepted`, and the last rows of `runs/<year>.md` in `~/Clawic/data/analysis/` (paths and formats in `memory-template.md`). Counts are meaningless without the previous ones.

**Contents:** [What A Run Row Holds](#what-a-run-row-holds) · [Comparability](#comparability) · [Recurrence](#recurrence) · [Acceptance](#acceptance) · [False-Positive Rate](#false-positive-rate) · [Metrics Worth Trending](#metrics-worth-trending) · [Cadence](#cadence) · [The Human Report](#the-human-report) · [Write It Down](#write-it-down)

## What A Run Row Holds

Date, mode, phases actually run, counts by severity, findings opened, findings closed, fixes applied with their verification result, and phases skipped with the reason. The last field is what keeps history honest: a run that skipped three phases and found nothing is not a clean run, and six months later nobody will remember why the numbers dipped.

Full evidence blobs do not belong in the run row — keep them for one cadence period at most, then drop to counts. The row is a time series; the details live in the findings and the artifacts.

## Comparability

Two runs are comparable only if they covered the same phases with the same thresholds. Anything else is a different measurement wearing the same units.

- Record the mode and the phase list on every row.
- When a threshold changes (`memory_budget_mb`, `secret_rotation_days`, `max_findings_shown`), write the change as its own row with no counts. A step in the trend that came from a threshold edit, unlabelled, is a trap for whoever reads the graph next quarter.
- Never compare a targeted run against a full one. Report the phase list in the header, and refuse the comparison out loud rather than producing a misleading delta.

## Recurrence

| Pattern | Meaning | Action |
|---|---|---|
| Same finding, 2 consecutive runs | The fix did not stick, or it was never a real problem | Escalate one severity and fix the generator, or move to `## Accepted` (SKILL.md Rule 6) |
| Same finding, 3 times in 90 days | Systemic: something recreates it | Fix the source — the template, the habit, the job that rewrites the file — and say so in the finding |
| Whole category clean for 3 consecutive full runs | The checks are cheap insurance, not a live area | Drop that phase to half cadence; do not delete the checks |
| A finding that appears only after a specific event (a deploy, a machine change) | Not periodic, conditional | Attach the trigger to the finding; run the phase on that event rather than on the calendar |
| A finding closed and reopened three times | The two sides disagree about what "fixed" means | Write the verification criterion into the finding itself |

"Fix the generator" is the whole point of tracking. The instance is a symptom; the template that keeps producing it is the finding.

## Acceptance

An acceptance row has five fields and is useless without all five: the rule or check it suppresses, the **scope** (a path glob, a slug, a job name — never "that file that day"), the reason in the user's words, the date, and a review date.

- Default review interval: 90 days, or `secret_rotation_days`, or the credential's own expiry — whichever is soonest.
- On every run, acceptances past their review date are **raised, not honored**. One line each: "accepted on <date>, review due — still true?" That single behavior is what stops the acceptance list from becoming a permanent blind spot.
- Scope creep is the failure mode: an acceptance written for one file that ends up matching a directory. When a suppression starts hiding more than three findings, it is too broad and gets split.
- An acceptance is never a deletion. The check keeps running; the finding is filed rather than shown, and it still appears in the counts as suppressed.

## False-Positive Rate

```
fp_rate = suppressed_in_this_run / total_findings_in_this_run
```

Above ~30%, the check set is miscalibrated for this setup and the audit is training the user to skim. The fix is to tighten the rule that produces the noise — narrow the pattern, raise the threshold, add the second signal (`secrets.md` entropy rule) — not to suppress more instances. Record the rate in the run row; it is the audit auditing itself.

The opposite signal matters too: a run that finds nothing, several times in a row, in a setup that is actively changing, usually means the checks are not reaching the changing part. Compare what changed in the workspace against what the phases cover.

## Metrics Worth Trending

Five numbers, all cheap, all comparable across months:

| Metric | Computed | Healthy direction |
|---|---|---|
| Open criticals | count in `## Open Findings` | 0, always; anything else is the headline |
| Mean age of open findings | today − opened date, averaged | Flat or falling; a rising mean means the list is a graveyard |
| Recurrence rate | findings seen in ≥2 of the last 3 runs ÷ total | Falling |
| Always-loaded token tax | SKILL.md Rule 8 formula | Falling or explained |
| Monthly spend | `## Spend` row with currency (`cost.md`) | Tracking usage, not drifting from it |

Four data points make a trend; two make a line. Say "two runs" when that is what there is.

## Cadence

Frequency follows exposure and change rate, not enthusiasm. Pick the row that matches, then let the user override — the choice becomes `audit_cadence` and its `## Due` rows.

| Setup | Full audit | Quick check | Why |
|---|---|---|---|
| Unattended automations, external triggers, or credentials with real blast radius | monthly | weekly | Unattended paths fail silently and pay for themselves the first time |
| Daily interactive use, few or no schedules | quarterly | monthly | Change is observed as it happens |
| Occasional use, single machine, no automations | twice a year | — | The audit costs more than the risk otherwise |
| Anything shared with other people or machines | monthly | weekly | Blast radius is not yours alone |

Event-driven runs beat calendar ones for three cases, and they go in `## Due` as conditions rather than dates: after adding an integration or a credential, after granting a new permission, and after any incident.

Every accepted cadence becomes a `## Due` row with its last run and next due date, checked at session start and stated in one line when overdue — a statement, not a question.

## The Human Report

Written for a person who was not in the session, at the end of a full audit or on request. It is an artifact, not a chat message: `~/Clawic/data/analysis/artifacts/health-report-<yyyy-mm>.md`, with its `## Boxes` line in the same turn.

Contents, in this order and nothing else: the date and which phases ran; the headline (open criticals, or "none"); what changed since the last report (opened, closed, still open with age); the three numbers from the metrics table that moved; what needs a human decision, with the exact action for each; and what was accepted and when it comes up for review.

What it must not contain: raw evidence dumps, secret values or fragments, a complete list of forty findings, or praise. A report longer than one screen is a report that gets filed unread — the details are in the findings and the artifacts, which is what those boxes exist for.

## Write It Down

Same turn as closing the run:

- The run row with counts, phases run, phases skipped and why, fixes and their verification results → `runs/<year>.md`.
- Findings that survived → `## Open Findings`, each with its first-seen date so the age metric works.
- New acceptances and any review-date changes → `## Accepted`.
- Updated cadences, plus the date this run satisfied → the matching `## Due` rows.
- Trend numbers worth keeping and any threshold change → `## System Baseline`.
- The written report → `artifacts/health-report-<yyyy-mm>.md`, plus its `## Boxes` line.
