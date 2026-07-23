---
name: triage
slug: triage
version: 1.0.1
description: >-
  Prioritizes incoming tasks into P0-P3 by cost of delay and learns the user's real priority rules from their corrections.
  Use when tasks compete, when deciding whether to interrupt current work, or when the user says urgent, ASAP, no rush, or reorders the queue.
homepage: https://clawic.com/skills/triage
changelog: Sharper prioritization thresholds
metadata:
  clawdbot:
    emoji: 🚦
    displayName: "Triage / Task Prioritization"
    configPaths: ["~/clawic/triage/"]
---

Classify incoming work into P0-P3 by cost of delay, route it, and learn the user's real priority rules from their corrections. Learned rules persist in `~/clawic/triage/` (created on first confirmed rule; nothing is stored without the user's explicit yes). Signal catalog: `signals.md`. Learning protocol and memory format: `patterns.md`.

## When To Use

- Several tasks compete and you must pick an order
- A new request arrives mid-task and you must decide whether to interrupt
- The user uses priority language: "urgent", "ASAP", "drop everything", "no rush", "when you have time"
- The user reorders or overrides your queue — that is training data, not just an instruction
- Not for effort estimation or roadmap planning: triage decides *when*, not *how long*

## Quick Reference

| Situation | Play |
|-----------|------|
| "Drop everything and X" | Interrupt now, but name what gets paused: "Pausing Y (was P1) — resume after?" |
| Two tasks both claim P0 | Escalate sequencing to the user — two "drop everything" cannot both be true |
| Sounds urgent, but no deadline and no accumulating damage | Cap at P1; one closed question: "Is anyone blocked right now?" |
| Mixed signals ("urgent but no rush") | Ask — the contradiction is the signal; never average to P1.5 |
| Deadline exists but looks far | Compute slack (Core Rules #2); slack under one working day → P1 today |
| User corrects your priority | Record it; on the 2nd same-direction correction, propose a standing rule (`patterns.md`) |
| Confirmed rule matches a new task | Apply it and cite it: "Deploy issue → P0 (your standing rule)" |
| New fact arrives (deadline revealed, blocker cleared, scope change) | Re-triage the whole queue, not just the new item |
| Else | Run the level tests below top-down; torn between adjacent levels → Core Rules #3 |

## Priority Levels

Each level has a boundary test. Run the tests top-down; first yes wins. Test states, not tone.

| Level | Test (yes → this level) | Response | Examples |
|-------|-------------------------|----------|----------|
| P0 | Does damage accumulate every minute this waits? | Interrupt current work now | Production down, active breach, data loss in progress |
| P1 | Is a person or deliverable stalled today? | Next work-unit boundary, same day | Blocked teammate, client waiting, due-today deadline |
| P2 | Dated or clearly valuable, but nobody stalls today? | Scheduled queue, with a date | Reviews, planning, deadlines beyond today |
| P3 | Would anyone notice if this never happened? (barely) | Backlog, batch-processed | Ideas, "someday", minor optimizations |

## Core Rules

1. **Priority = cost of delay, never effort or loudness.** A hard task nobody waits on is P2; a 5-minute unblock for a stalled teammate is P1. Tie-break within a level: highest cost of delay per unit of effort first (WSJF, Reinertsen); when costs look equal, shortest job first.
2. **Deadline urgency is slack, not calendar distance.** Slack = working time until deadline − remaining work. Friday deadline + 3 days of work, seen on Wednesday = negative slack = P1 today. Count working hours only: a Monday-9am deadline seen Friday afternoon has near-zero slack. Recompute whenever scope grows.
3. **Misclassification is asymmetric — up is cheap, down is expensive.** A false P0 costs one interruption; a missed P0 costs the whole damage window. Torn between adjacent levels *with* concrete urgency evidence → take the higher. No evidence, just tone → take the lower and label it: "Queued as P2 — say the word if it's blocking."
4. **Ask only at the P0/P1 boundary.** One closed question there beats a wrong guess. At P2/P3 the stakes don't cover the interruption cost — pick, state the assumption, move on.
5. **Learn only through the ladder: observe → propose → confirm.** One correction = record. Two same-direction corrections = propose ("Should deploy issues always be P0?"). Apply automatically only after an explicit yes, and cite the rule when applying it. Never silently internalize — full ladder and decay in `patterns.md`.
6. **P0 is rare by construction.** If over a week more than ~1/3 of items land P0/P1, the bar drifted — re-anchor on the level tests instead of inventing intermediate levels. When everything is urgent, nothing is.
7. **Re-triage on state change, not on a timer.** Triggers: deadline revealed or moved, blocker cleared, scope change, new P0. Each one re-sorts the whole queue; a queue sorted on stale facts is a wrong queue.

## Queue Discipline

- P0 interrupts mid-task. P1 waits for the current unit of work to finish — dropping a half-done change leaves broken state, which is its own incident. Exception: if current work is P2/P3, switch at the next safe point.
- Order within a level: confirmed user rules first, then ascending slack, then arrival order.
- Announce changes, not steady state: "New P0 — pausing X. Queue now: [incident, review, docs]." Silent reshuffles destroy the user's trust in the queue.
- A dated P2 gets scheduled the moment it enters the queue. A P2 that "ages into" a P0 was a triage failure, not an escalation — log it as a correction against yourself.

## Output Gates

Before emitting a priority or a reordered queue:

- Is every P0 justified by accumulating damage or a level test — not by tone, caps, or sender seniority?
- Did I check `~/clawic/triage/patterns.md` for a confirmed rule before falling back to defaults?
- Am I auto-applying any rule the user never confirmed? If yes → propose instead.
- If queue order changed, did I say so and name what got bumped?

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Sender rank as urgency | An exec's "someday idea" outranks nothing; rank predicts importance, not cost of delay | Run the level tests; if the user consistently overrides for a sender, learn that as a rule |
| Newest-task pull | Recency feels urgent; arrival time is not cost of delay | Slot by test result, re-sort, report the position |
| Averaging mixed signals | "Urgent but no rush" averaged to P1.5 satisfies neither reading | Ask — the contradiction is the signal |
| Escalating on repetition | A third mention is frustration, not new urgency | Acknowledge, give queue position and expected start |
| Demoting polite requests | "Whenever you get a chance" from some users means "today" | Check for a date before demoting; calibrate per sender (`signals.md`) |

## Where Experts Disagree

- **Interrupt-driven vs. batch triage.** Batch (fixed triage passes) wins when true P0s are rare and deep work dominates; interrupt-driven wins for on-call contexts. Pick per user context and tell the user which mode you're running.
- **FIFO vs. WSJF within a level.** FIFO is predictable and reads as fair to requesters who watch the queue; WSJF delivers more value when only outcomes are visible. Perceived fairness is a real cost — choose by who is watching.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):

- `task-list` — maintaining the queue itself once triage has assigned levels
- `incident-response` — when a P0 is a production incident: triage decides the interrupt, that skill runs the handling
- `time-blocking` — protecting focus time that P2/P3 must not interrupt
- `inbox` — processing message streams before items become triaged tasks

---

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/triage.
