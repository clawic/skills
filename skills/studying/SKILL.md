---
name: studying
slug: studying
version: 1.0.2
changelog: Complete rewrite with real study protocols
description: Plans and runs studying with retrieval practice, spaced review formulas, and exam countdown protocols. Use when a user prepares for courses, tests, or certifications, asks how to memorize material, or wants a revision schedule. Learns which techniques work for this student and persists them across sessions.
homepage: https://clawic.com/skills/studying
metadata:
  clawdbot:
    emoji: "📖"
    displayName: Studying
    configPaths:
    - ~/clawic/studying/
---

Session history and per-student preferences persist in `~/clawic/studying/memory.md` (create on first use, see Preference Memory).

## When To Use

- A user has an exam, test, or certification on a date and needs a plan to get there
- A user asks how to learn or memorize specific material (chapter, deck, problem set, lecture)
- A user reports "I studied hard but forgot everything" or failed a practice test
- Building a weekly revision schedule across multiple courses
- Mode: advise. This skill coaches a human student; it does not do the assignment for them (that is `homework` territory)
- Not for designing flashcards themselves (`flashcards`, `anki`) or generating practice tests (`exam`)

## Quick Reference

| Situation | Play |
|---|---|
| Exam 4+ weeks out | Successive relearning: learn to criterion now, relearn in 3+ spaced sessions, gap = 10-20% of days remaining (→ Core Rule 2) |
| Exam in 5-7 days | One full timed past paper or self-test day 1 to locate gaps; spend remaining days on missed items only, 1-day gaps |
| Exam tomorrow | Retrieval-only sprint on highest-yield gaps, stop new material, protect a full night of sleep (→ Core Rule 6) |
| Fact-heavy material (vocab, anatomy, dates, law) | Flashcards with successive relearning; cap new cards so daily reviews stay under ~30 min |
| Problem-based material (math, physics, coding) | Block each problem type until 2-3 correct solo solves, then interleave types in every later session |
| Conceptual material (theories, mechanisms, essays) | Closed-book explanation: write or say the concept from memory, mark every gap, check source, repeat next session |
| Student says "it feels easy now" | Fluency illusion until proven: schedule a delayed self-test 1-2 days out before trusting it |
| Student failed a practice test | Diagnose per question: never-encoded vs forgot vs misread; each gets a different fix (→ Diagnosing Misses) |
| Multiple courses competing | Allocate time by (exam weight x current weakness), not by comfort; weakest-highest-stakes course gets the first fresh hour |
| Anything else | Default loop: 10 min recall of last session, new material with self-generated questions, close with a 5-question self-test |

## Core Rules

1. Retrieval beats re-exposure: at least 50% of any session is closed-book recall (self-test, blank-page dump, problem solving). Check: if the session log shows pages read but zero questions answered, the session failed. One week out, tested material is recalled at roughly 1.5x the rate of restudied material (Roediger and Karpicke).
2. Spacing formula: review gap = 10-20% of time until the test (Cepeda). Exam in 30 days: gaps of 3-6 days. Exam in 10 days: gaps of 1-2 days. For multi-month horizons the optimal ratio drifts down toward 5-10%; when unsure, choose the longer gap because too-long beats too-short at test time.
3. Successive relearning protocol (Rawson and Dunlosky): first session, practice each item until 3 correct recalls; every later session, until 1 correct recall; minimum 3 relearning sessions before the exam. Dropping an item after one correct answer is the single most common scheduling error.
4. Interleave only after acquisition: block a new problem type until 2-3 correct unassisted solves, then mix types so the student must pick the method, not just execute it. In Rohrer's math studies interleaved practice roughly doubled delayed-test scores versus blocked practice while feeling worse during practice.
5. Judge learning only after a delay: confidence rated immediately after studying is inflated by short-term fluency (Bjork's desirable difficulties). Rate an item "known" only if recalled cold at the start of a later session.
6. Sleep is part of the schedule: memory consolidation happens during sleep, so the night before the exam is study material too. If the plan requires trading sleep for new content within 24h of the exam, the plan is wrong; cut lowest-weight topics instead.
7. Low-utility techniques are pre-processing only: rereading, highlighting, and summarizing rate low-utility in Dunlosky's technique review. Allowed for exactly one pass whose sole output is a question list to feed retrieval practice.

## Session Protocol

1. Open with recall, not review: 5-10 min writing everything remembered from last session, blank page, book closed. Then check and mark gaps.
2. New material pass: for every section, generate 2-5 test questions (definition, why, applied case) and log them; questions are the session's durable artifact, highlights are not.
3. Close the loop: answer today's new questions plus all questions missed last session, cold. Anything missed twice gets flagged for the next session's opening.
4. Log one line to memory.md: date, topic, minutes, retrieval hit rate (correct / attempted). Hit rate below 60% next session means gaps are too long or items too big: split items or halve the gap. Above 90% two sessions running means gaps are too short: lengthen toward the Rule 2 ceiling.
5. Session length: stop when retrieval accuracy visibly degrades within the session, not at a fixed timer; log the duration where that happened and treat it as this student's default block length.

## Exam Countdown

- T-4 weeks: inventory all topics, weight by syllabus points; build spaced slots per Rule 2; start successive relearning on fact-heavy topics first because they need the most sessions.
- T-2 weeks: first full-length timed practice test under exam conditions (same time limit, no notes, same allowed tools). Score it, then re-plan: topics below ~70% get double slots, topics above 90% drop to maintenance (one retrieval pass per week).
- T-1 week: second timed test; practice the exam's actual format (essay outlines in essay courses, problem sets in problem courses). Format-mismatched practice inflates confidence without transferring.
- T-1 day: retrieval-only, no new topics, half-length day, full night of sleep. Prepare logistics (location, materials, ID) the evening before, not the morning of.
- Post-exam, 10 min: log to memory.md what the exam actually tested versus what the plan predicted; this calibrates the next countdown.

## Diagnosing Misses

For each miss on a self-test or practice exam, classify before fixing:

- Never encoded (no memory of ever seeing it): coverage hole; add to new-material queue, not to review queue.
- Encoded but forgot (recognized the answer instantly on reveal): spacing failure; shorten gap for that item and re-enter relearning at 3 correct recalls.
- Knew it, misapplied it (right fact, wrong method or wrong question read): practice-format failure; fix with mixed timed sets, not more flashcards.
- Ran out of time: pacing failure; all further practice tests get a per-question time budget = total minutes / question count, enforced.

## Preference Memory

Persist in `~/clawic/studying/memory.md` with sections: Techniques (what worked, with evidence level), Schedule (proven block length and best time of day from session logs), Materials, Exams (post-exam calibration notes), Never (approaches that failed for this student twice). Promote an observation pattern -> confirmed after 2+ consistent signals; confirmed entries override this skill's defaults except Core Rules 1, 2, and 6, which are non-negotiable floors.

## Red Flags

| Signal (observable) | Suspicion | Action |
|---|---|---|
| Second consecutive night under ~4h sleep to study | Sleep deprivation degrading the memory being built | Stop the plan, sleep first; replan with cut topics |
| Escalating stimulant use (doubling caffeine, borrowed prescription drugs) | Dependence or dangerous dosing | Do not optimize around it; flag it and route to a clinician |
| Panic symptoms during study or exams (racing heart, blanking, nausea) | Anxiety condition beyond technique fixes | Suggest campus counseling or a clinician; keep sessions short meanwhile |
| Skipped meals across multiple days to extend study time | Disordered eating pattern | Rebuild schedule around fixed meals; persistent pattern goes to a clinician |

Anything in this table suspends the protocols above: route to a clinician or counselor before resuming optimization.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Rereading until it "feels familiar" | Familiarity is recognition, not recall; it evaporates at the exam | Convert every reread urge into a closed-book question attempt |
| Massing all review into the final 48h | Cramming survives days, not the follow-on course or cumulative final | Rule 2 spacing; final 48h is retrieval-only maintenance |
| Reading worked solutions and nodding along | Recognizing a solution is not generating one | Cover the solution, re-derive solo; count only unassisted solves |
| Making flashcards or notes all session | Production feels like progress but contains zero retrieval | Cap creation at 30% of session; the rest is answering |
| Studying the favorite subject first while fresh | Comfort allocation; the weak high-stakes course gets tired hours | Weakness x weight ordering from Quick Reference |
| Trusting confidence right after studying | Immediate judgments of learning are inflated by fluency | Only delayed cold recall marks an item known (Rule 5) |
| All-nighter before the exam | Loses the consolidation night and degrades exam-day retrieval | Cut topics, keep the sleep (Rule 6) |
| Untimed, open-note practice tests | Trains a different task than the exam | Exam conditions from the first practice test onward |

## Related Skills

- `exam`: generate the practice tests and timed simulations this skill schedules
- `flashcards` / `anki`: card design and deck mechanics for the fact-heavy tracks
- `spaced-repetition`: deeper scheduling theory when tuning gaps beyond Rule 2
- `deep-work`: protecting the distraction-free blocks these sessions run inside

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/studying.
