---
name: plan
slug: plan
version: 1.0.1
description: >-
  Decides when to plan vs execute directly, sizes plan depth to risk, and improves planning
  through outcome tracking. Use when a task has multiple steps, irreversible actions, unclear
  success criteria, or the user asks for a plan before work starts.
homepage: https://clawic.com/skills/plan
changelog: Deeper planning heuristics throughout
metadata:
  clawdbot:
    emoji: 🗺️
    displayName: Plan
    configPaths: ["~/clawic/plan/"]
---

## When To Use

Use when a task has multiple steps, irreversible actions, unclear success criteria, or the user asks for a plan before work starts. Skip planning docs for work done before successfully and fully reversible (L0) — but always make the risk decision first. Planning records live in `~/clawic/plan/` (outcome log and per-type defaults).

---

## Quick Reference

| Situation | Do this |
|-----------|---------|
| Done before, fully reversible | Execute directly (L0) |
| Single deliverable, ≤30 min, reversible | Think through steps, no doc (L1) |
| Multi-step, >30 min, or any irreversible step | Bullet plan, share with human (L2) |
| Dependencies across components, or spans >1 day | Milestone plan with checkpoints (L3) |
| High stakes AND novel task type | Full plan, human validation before step 1 (L4) |

One irreversible step anywhere forces at least L2. When uncertain, plan.

---

## Core Rules

1. **A plan's value is the risk decision, not the step list.** Force it early: what single assumption could invalidate the whole approach, and can it be tested cheaply first? A to-do list with no risk ordering adds ceremony, not safety.
2. **Match planning effort to failure cost.** Depth = the highest level any single signal triggers, not an average.
3. **Irreversibility dominates.** One irreversible step forces at least L2 regardless of other signals — you cannot iterate your way out of a deleted database or a sent email.
4. **Learn per task type.** Track which depth and strategy actually work: strategy templates in `strategies.md`, recording and learning loop in `outcomes.md`.

---

## The Planning Decision

Before executing, scan for signals:

| Signal | One-shot OK | Plan needed |
|--------|-------------|-------------|
| Task done before successfully | ✅ | |
| Clear single deliverable | ✅ | |
| Reversible if wrong | ✅ | |
| Multiple components | | ✅ |
| Dependencies between steps | | ✅ |
| Any irreversible step | | ✅ |
| Touches production data or external users | | ✅ |
| Ambiguous success criteria | | ✅ |
| Estimated >30 min work | | ✅ |

**Irreversibility dominates.** One irreversible step anywhere forces at least L2 regardless of other signals — you cannot iterate your way out of a deleted database or a sent email.

**Reversible vs recoverable** — the distinction that decides the column: reversible means undo restores the prior state (git revert); recoverable means a good state is reachable at a cost (restore last night's backup, lose a day of writes). Recoverable-at-cost counts as "plan needed", not "reversible".

**Default:** when uncertain, plan. The cost is asymmetric — a bullet plan costs minutes; a failed one-shot costs the redo plus the cleanup plus the trust.

---

## Plan Depth Levels

| Level | Trigger | Format |
|-------|---------|--------|
| L0 | Done before successfully, fully reversible | Execute directly |
| L1 | Single deliverable, ≤30 min, reversible | Think through steps; no doc |
| L2 | Multi-step, or >30 min, or any irreversible step | Bullet plan shared with the human |
| L3 | Dependencies between components, or spans >1 day | Milestone plan with checkpoints |
| L4 | High stakes AND novel (this task type never done) | Full plan; human validation before step 1 |

Depth = the highest level any single signal triggers, not an average. A 20-minute task with one irreversible step is L2, not L1.

---

## Plan Format (L2-L4)

```
📋 Plan: [goal]

Why planned: [the signal that triggered planning — one line]

Steps:
1. [step] — [observable output that proves it is done]
2. [step] — [observable output]
3. [step] — [observable output]

Riskiest assumption: [what invalidates the approach] — tested in step [N]
Irreversible steps: [numbers] — rollback: [how] (or "none")

Estimate: [low–high range]
Validation: [none | human approves before step N]

Ready to start?
```

Rules that make the format work:

- **3-7 steps.** More than 7 means wrong altitude: group into milestones and plan only the first milestone in step detail.
- **A step you cannot attach a check to is not a step, it is a hope.** "Investigate X" becomes "decision doc: X vs Y, with the pick". If no check is possible, convert it to a spike (`strategies.md`).
- **Order by information, not convenience.** The step most likely to invalidate the plan goes as early as dependencies allow. Migration example: write and test the rollback script as step 1, not last — if rollback is impossible, you want to know before touching data.
- **Detail decays.** Steps written past the first unknown are speculation you will rewrite. Plan in steps to the first checkpoint; milestones beyond it.
- **Estimates are ranges.** If the high/low ratio exceeds 3x, that is not an estimate, it is an unknown — spike first. 2-8h (4x) → spike; 3-6h (2x) → plan.

---

## Executing Against The Plan

- Approval covers what is written. At L3+, a materially different execution needs re-approval, not a retroactive mention.
- **Replan trigger:** when 2 consecutive steps deviate from plan (skipped, reordered, or output differs from stated), stop and replan instead of patching step-by-step. One deviation is noise; two consecutive means the model of the task is wrong.
- Record every deviation in the outcome log — deviations are the raw material for next time's plan (`outcomes.md`).

---

## Validation Learning

Track per task type whether human validation is still needed:

```
### Auto-Execute (validation waived by human)
- refactor/small: L2 [streak: 7]

### Validate First
- migration/data: L4 [always — irreversible]

### Learning
- api/integration: L2, streak 3/5 toward auto-execute proposal
```

**Promotion rule:** after 5 consecutive successful validated runs of a plan type, ask: "Should I auto-start [type] plans without validation?" One failure resets the streak and restores validation. The bar is higher than depth demotion (below) because this removes a human safety net, not just ceremony.

---

## Outcome Tracking

After every L2+ task, append a record to `~/clawic/plan/outcomes.md` — full record format and analysis procedure in `outcomes.md`. L0/L1 tasks are not logged (logging trivial work drowns the signal), with one exception: an L0/L1 that failed gets a record, because it is evidence the type needs promotion.

---

## Strategy Learning

Different goals fail differently; strategy choice and combinations live in `strategies.md`. When a strategy fails for a task type, the log entry must say why it failed — "Parallel → merge conflicts" teaches; "Parallel → didn't work" does not.

---

## Plan Refinement

Learning rates are asymmetric because evidence costs are asymmetric:

- **Promote depth after ONE failure** attributable to plan depth — and name what the deeper level would have caught. If you cannot name it, it was not a planning failure and no promotion happens.
- **Demote depth after 3 consecutive successes** where the extra depth went unused. Observable signal: plan sections never consulted during execution.
- **Auto-execute needs 5 consecutive successes** (→ Validation Learning above).

---

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Planning as procrastination: detailing steps past the first unknown | That detail is speculation; you will rewrite it after the unknown resolves | Steps to the first checkpoint, milestones beyond |
| Steps written as activities ("investigate X") | No observable output means no done-check; plans drift silently | Every step names the artifact or check that proves completion |
| Uniform depth for every task | L4 on trivial work trains the human to skim-approve; L1 on novel work ships failures | Depth table + learned per-type defaults |
| Sticking to a plan reality has contradicted | A plan is a forecast, not a commitment; adherence becomes sunk cost | 2-consecutive-deviation replan trigger |
| Point estimates | Read as commitments and hide uncertainty | Range; ratio >3x → spike first |
| Executing beyond approved scope without saying so | Approval covers the written plan only; burned trust raises validation on everything after | Announce deviations; re-approve at L3+ |
| Logging only failures | Success streaks are what earn auto-execute; unlogged successes mean validating forever | Log every L2+ outcome, success included |

---

Empty tracking files = early stage, not a defect: execute, record, learn.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/plan (install if the user confirms):

- `decide` - Choose between options inside a plan step
- `escalate` - Ask-vs-act boundaries beyond planning scope
- `self-improving` - General execution lessons; plan keeps only planning lessons
- `memory` - Long-term context and user continuity beyond planning records

## Feedback

- If useful, star it: https://clawic.com/skills/plan
- Latest version: https://clawic.com/skills/plan

---

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/plan.
