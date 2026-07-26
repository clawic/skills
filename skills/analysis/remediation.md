# Fixing — What May Be Touched, In What Order, And How To Prove It Worked

**Before fixing anything**, read `config.yaml` for `autofix_policy` and `## Open Findings` in `~/Clawic/data/analysis/memory.md` for what is already in flight. Fixing a finding another session opened, without closing its row, produces a permanent phantom on every future report.

**Contents:** [The Reversibility Test](#the-reversibility-test) · [Policy](#policy) · [Order Of Operations](#order-of-operations) · [Batch Size](#batch-size) · [Verification](#verification) · [Recipes](#recipes) · [Never Auto-Fix](#never-auto-fix) · [When A Fix Breaks Something](#when-a-fix-breaks-something) · [Write It Down](#write-it-down)

## The Reversibility Test

A fix may be applied without asking only if all four hold. Three out of four is a proposal.

1. **Inverse exists and is recorded** — the old value, the old mode, or a copy of the file, written down *before* the change (SKILL.md Rule 5).
2. **Blast radius is one thing** — one file, one setting, one process. Anything that changes behavior for other people or other machines is out.
3. **Detection can be re-run** — the same check that found it can prove it is gone, now, without waiting for an event.
4. **No data leaves and none is destroyed** — nothing deleted, nothing transmitted, nothing rotated at a third party.

## Policy

| `autofix_policy` | Behavior |
|---|---|
| `propose` (default) | Nothing is written. Every finding ships with the exact command and its inverse |
| `safe-only` | Fixes passing all four tests are applied and reported in one line each; everything else is proposed |
| `ask-each` | Every fix, including safe ones, is confirmed individually before it runs |

`safe-only` never escalates: a fix that fails the test is proposed even when the user has been approving similar things all session. Consent to one fix is not consent to a class.

## Order Of Operations

Across findings, not within one. The order exists because early phases destroy the evidence later ones need, and because some fixes are worthless until an earlier one lands.

1. **Stop the bleeding** — a session spending money, a job in a retry storm, a process filling the disk. Nothing else matters while one of these is running (`sessions.md`).
2. **Rotate exposed credentials** — before any file is touched, so the add-timestamp and the history survive for the access-log check (`secrets.md`).
3. **Close authority holes** — allowlist entries, egress, unattended grants. Do this before restoring broken automations, or you re-enable a job that runs with the grant you were about to remove (`permissions.md`).
4. **Restore function** — jobs, integrations, tokens. Now the system works and it is not dangerous.
5. **Repair data paths** — read order, dangling references, contradictions (`agent-memory.md`).
6. **Reclaim** — retention, pruning, consolidation, archiving. Last, always: this is the only class that destroys evidence.

Within a class, fix by severity, then by cheapness.

## Batch Size

| Change class | Changes before verifying |
|---|---|
| Destructive or irreversible | 1 |
| Config or permission changes | 1 — behavior changes are unattributable in batches |
| Reversible file edits of the same kind (modes, retention lines, index entries) | up to 5 |
| Pure additions (an index line, a `## Boxes` entry) | the whole set |

Rationale rather than superstition: if two changes land and something breaks, isolating the cause costs more than the second change saved.

## Verification

A fix is finished when the original detection has been re-run and comes back clean. Not "the file looks right" — the same command, the same arguments, and a different result.

| Fix | Verification |
|---|---|
| Permission tightened | Re-stat; the mode is the intended one |
| Credential rotated | An authenticated call with the old value returns 401/403 |
| Credential replaced with a pointer | Re-scan the file: zero prefix hits, and the consumer still works |
| Retention added | The rule exists *and* the next natural boundary has been simulated or reasoned through |
| Job schedule corrected | Print the next five fire times; they match intent (`scheduled.md`) |
| Dead-man's switch added | Force a miss in a scratch context and confirm the alert path fires |
| Index repaired | The set difference in both directions is empty (`workspace.md`) |
| Lock broken | The lock is gone and the job it blocked ran once |
| Allowlist narrowed | The pattern no longer matches the dangerous invocation, and the ordinary one still works |

When verification is impossible, say so and downgrade to a proposal. "It should be fixed now" is how the same finding returns in two runs and gets escalated by Rule 6.

## Recipes

Each row is finding class → fix → inverse → verification. Nothing here is destructive except where marked.

| Finding | Fix | Inverse | Auto? |
|---|---|---|---|
| Key file world-readable | Set mode 600 (700 on its directory) | Restore the recorded mode | yes |
| Credential value in a config file | Replace with `<kind>:<locator>`, put the value in the store the setup uses | Recorded pointer → original line, from the store | no — the store write and the rotation are the user's |
| Credential in git history | Rotate first, then propose a history rewrite | None; history rewrite is one-way for every clone | never |
| `.env` tracked in the repo | `git rm --cached`, add to ignore rules; rotate anything it contained | Re-add the path | no |
| Orphan file | Add its index line with a read condition | Remove the line | yes |
| Dangling reference | Repair the path, or remove the pointer | Recorded original line | yes |
| Contradictory entries | Keep the newer, delete the older, note the replacement | The deleted line, recorded | no — a wrong choice silently changes behavior |
| Conditional read order | Rewrite as unconditional, index-driven | Recorded original text | no — it changes what the agent does every session |
| Unbounded job output | Add rotation with size and count | Remove the rule | yes |
| No dead-man's switch | Add a success report plus an absence alert | Remove it | no — it creates an external registration |
| Job in the DST window | Move outside it, or express in UTC | Recorded original schedule | no |
| Overlapping job | Add a lock with a max age, or raise the interval | Remove the lock file mechanism | no |
| Stale lock | Break it, recording holder and age | Recreate is meaningless; record instead | yes |
| Stuck or zombie session | Capture evidence, then terminate | None | never |
| Broad allowlist entry | Replace with the narrower form | Recorded original entry | no |
| Expiring token | Reissue at the provider, update the pointer, update the expiry | Old token, if still within its life | no |
| Bloated always-loaded file | Move depth behind an explicit read; leave a pointer | Recorded original file | no — it changes behavior |
| Volatile line breaking the cache prefix | Move it below user content or out of the always-loaded set | Recorded original position | yes, when the line is not load-bearing |
| Unused skill or grant | Propose removal with what it enables | Reinstall or re-add | no |

## Never Auto-Fix

Regardless of `autofix_policy`: rotating or revoking a credential at a provider, rewriting git history, force-pushing, deleting user content of any kind, killing sessions or processes, changing anything on a remote host, uninstalling anything, editing the agent's own permissions, and sending anything to a third party. Each one is proposed with the exact command, its blast radius, and its inverse where one exists.

## When A Fix Breaks Something

1. Apply the recorded inverse immediately — this is why it is recorded before, not after.
2. Verify the original symptom is back and the new one is gone; if only the second is true, the inverse was incomplete and that is now the priority.
3. Write the failed fix and its cause into the run row: a fix that broke something is worth more to the next session than ten that worked.
4. Re-open the finding with severity unchanged and a note that the obvious fix does not work here. Downgrading it because it resisted is how it disappears.

## Write It Down

Same turn as the fix:

- Every fix attempted — finding id, what changed, the inverse, the verification result, and whether it held → the fix rows of `runs/<year>.md` (`memory-template.md`).
- Findings closed → removed from `## Open Findings`, with the closing date carried into the run row.
- A repair procedure worth repeating, with its verification → `~/Clawic/data/analysis/artifacts/fix-<kebab>.md`, plus its `## Boxes` line.
- A finding the user declines to fix → `## Accepted`, with the reason and a review date (SKILL.md Rule 7), never silently dropped.
