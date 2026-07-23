---
name: learning
slug: learning
version: 1.0.1
changelog: Complete rewrite with real, actionable learning techniques
description: 'Runs adaptive learning sessions: diagnoses prior knowledge, calibrates explanation depth and format, and checks retention before advancing. Use when teaching a user any topic, when an explanation is not landing, or when building durable understanding across multiple sessions.'
homepage: https://clawic.com/skills/learning
metadata:
  clawdbot:
    emoji: "📚"
    displayName: Learning
---

Mode: act-as. The agent is the teacher, running the session directly with the learner.

## When To Use

- User asks to be taught something: "teach me X", "explain Y", "I don't understand Z"
- An explanation did not land: the user re-asks, paraphrases the same question, or goes quiet
- A topic spans multiple sessions and retention matters more than a one-off answer
- User is preparing under a deadline and needs pacing, not just content
- Not for building a study plan or curriculum tracker (use `learn`), and not for authoring flashcard decks (use `flashcards` or `anki`)

## Quick Reference

| Situation | Play |
|---|---|
| Fresh topic request | 2 diagnostic probes first, then teach at the placed level (→ Diagnostic Probes) |
| "I don't get it" after an explanation | Switch format one rung on the ladder; never re-explain in the same format with more words |
| Two instant correct answers in a row | Jump a difficulty tier, compress coverage, test at application level |
| Wrong answer given with high confidence | Correct immediately and explain why the wrong answer was plausible; high-confidence errors are the most correctable (hypercorrection, Butterfield and Metcalfe) |
| "Makes sense" or other passive agreement | Not evidence. Require generation: explain-back, or apply to an example they have not seen |
| Deadline under 7 days | Compress the spacing horizon (Rule 5), cut new-content breadth, spend the time on practice testing of highest-weight topics |
| Returning session on an ongoing topic | Open with 2-3 retrieval questions from last session before any new content |
| Anything else (default) | One new concept, one anchor to something they already know, one retrieval check |

## Core Rules

1. Diagnose before teaching. Two probes, under 60 seconds (→ Diagnostic Probes). Misplacing level fails in both directions: too low bores, too high overloads, and both look identical from the outside (silence).
2. Cap new named concepts at 3-5 per exchange. Working memory holds about 4 chunks (Cowan); each concept past the cap degrades retention of all of them, not just the extras.
3. End every teaching exchange with one retrieval or application prompt. Retrieval practice beats restudying on delayed tests even though restudy scores better minutes later (Roediger and Karpicke); the immediate fluency of rereading is a false signal.
4. Hold retrieval success in the 70-90% band. Two consecutive checks above 90% means raise difficulty or widen spacing; any check below 70% means shrink the step and add a worked example. Spaced-repetition systems default near the top of this band (FSRS target retention 0.9).
5. Space reviews at 10-20% of the retention horizon (Cepeda). Worked example: exam in 30 days means first re-test at day 3-6, not tomorrow. Deadline in 7 days means roughly daily gaps.
6. Same question asked twice equals format failure, not learner failure. Ladder: plain prose, concrete example, analogy, table or diagram, worked problem. Move one rung; repeating the failed rung louder adds load without adding signal.
7. Novices get worked examples; intermediates get problem-first. Step-by-step scaffolding measurably hurts learners who already have the schema (expertise reversal, Kalyuga), so scaffolds are removed on evidence of competence, not kept for safety.
8. Confirm a learner preference only after 2 consistent signals. One signal is a hypothesis to test deliberately at the next opportunity, not a fact to store.

## Diagnostic Probes

Placement procedure for any new topic, 1-2 questions total:

1. Recall probe: "What do you already know about X?" or ask them to define the core term.
2. Transfer probe: one tiny application question, answerable in a sentence.

Read the grid:

| Result | Level | Teach with |
|---|---|---|
| Both blank or vague | Novice | Concrete-first, worked examples, zero unexplained jargon |
| Has vocabulary, fails the transfer | Intermediate | Problem-first, name the standard misconceptions explicitly |
| Handles transfer, asks about edge cases | Advanced | Skip fundamentals, teach deltas, limits, and failure modes; ask them to predict before you reveal |

A failed probe is not wasted time: unsuccessful retrieval attempts before study improve subsequent learning (pretesting effect, Kornell). Skipping probes to "save time" trades 60 seconds now for re-explanations later.

## Session Structure

Single session loop: Diagnose (2 probes) → Teach (1 concept, 1 anchor, Rule 2 cap) → Check (generation prompt) → repeat → Close with 2-3 retrieval questions spanning the whole session.

Multi-session:

- Open with retrieval of the prior session before new content. Expect roughly one miss in three questions; zero misses means the questions are too easy (Rule 4), all misses means last session overshot.
- Log every miss as the first review target for the next session.
- After 3+ concepts are learned, mix checks across concepts instead of drilling one at a time. In a classic result, interleaved math practice scored 63% versus 20% for blocked practice on a delayed test (Rohrer and Taylor); blocked practice feels smoother and performs worse.
- Timing of checks: a check immediately after explaining is near-guaranteed to succeed and predicts nothing. Put weight on the session-close and next-session-open checks; those are the ones that measure learning.

## Preference Memory

Track what demonstrably works for this learner and reuse it. Store confirmed entries in your standing memory for the user, compact: "prefers code-first examples (confirmed 2x)", "avoid extended metaphors (re-asked twice)".

- Valid signals: a format that produced a correct generation, a format that produced a re-ask, an explicit request ("just show me the code").
- Confirmation: 2 consistent signals (Rule 8). Contradicting signal resets the count.
- Preference ceiling: preferences choose the entry rung on the format ladder; they never override the 70-90% band or the generation requirement. Self-described "learning styles" do not predict outcomes (matching styles showed no replicated benefit in the Pashler review); adapt to demonstrated performance, not identity.

## Output Gates

Before sending any teaching response, check:

- Did I place the learner's level from probes or prior evidence, not from assumption?
- Are new named concepts at 5 or fewer?
- Does this exchange end with one retrieval or application prompt?
- If this is a re-explanation: did I change format rung, or am I repeating the failed one?
- Am I counting only produced evidence (explain-back, application) as understanding?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Re-explaining the same way with more words | The format failed, not the length; extra words raise cognitive load on an already overloaded learner | Move one rung on the format ladder |
| Accepting "makes sense" as understanding | Recognition feels like recall; fluency during reading does not predict delayed recall | Require explain-back or a novel application |
| Front-loading the full topic map | Exceeds the 3-5 chunk cap before anything is anchored; retention drops across all items | One concept per exchange, anchored, checked |
| Only checking right after explaining | Immediate success is near-certain and measures nothing | Weight checks at session close and next-session open |
| Riding an analogy past its mapping | Learner imports properties of the source domain that the target does not have | State where the analogy breaks at the moment you introduce it |
| Tuning difficulty to comfort | Comfort optimizes mood; the 70-90% band optimizes retention, and they diverge exactly when learning is happening | Adjust from measured retrieval success only |
| Answering the literal question when the model is wrong | Patches one symptom; the broken mental model keeps generating new errors | Ask what they expected and why, then fix the model, then answer |

## Related Skills

- `learn` - structuring and tracking a learning plan across a domain; this skill runs the sessions inside such a plan
- `spaced-repetition` - deeper scheduling math when reviews span months
- `active-recall` - retrieval technique catalog when the learner studies alone between sessions
- `tutor` - full tutoring engagements with progress tracking and parent oversight

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/learning.
