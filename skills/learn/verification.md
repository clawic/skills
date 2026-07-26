# Verification — Proving It Was Actually Learned

Read before promoting a topic in `## Topics`, before declaring a stage done, and whenever confidence and results disagree. The Mastery Ladder itself lives in `SKILL.md`; this file is how each level is tested and how false confidence is caught.

**Contents:** [Designing the Test](#designing-the-test) · [Testing Each Level](#testing-each-level) · [Calibration](#calibration) · [Signals of False Confidence](#signals-of-false-confidence) · [Understanding vs Memorisation](#understanding-vs-memorisation) · [The Cold Re-Test](#the-cold-re-test) · [Failing Well](#failing-well) · [Certificates and Other Non-Evidence](#certificates-and-other-non-evidence)

## Designing the Test

A verification test is written **before** the material is studied where possible, and always by working from the exit test, never from the notes. A test built from notes measures whether the notes were read.

Five properties, all required:

| Property | Why | Violation looks like |
|---|---|---|
| Unseen | Anything practised measures practice | Re-running the exercises from the book |
| Unprompted | Real use does not say which method applies | "Using the chain rule, differentiate…" |
| Cold | Warm-up hides the retrieval failure that matters | Testing at the end of a study session |
| Whole-task | Sub-skills can all pass while the assembly fails | Grading five isolated drills and calling it done |
| Time-bounded | Unlimited time converts recall into re-derivation | "I got it eventually" |

Draw the content from `errors/<topic>.md` first: the learner's own misconception list is the highest-yield test bank available, and it is free.

## Testing Each Level

| Level | Test | Passing bar |
|---|---|---|
| Recognition | Multiple choice, matching | Not a bar — never promote on this |
| Recall | Blank-prompt production, no options | 4 of 5 items, cold |
| Application | An unseen problem of a known type, no hint about method | Solved unaided within the domain's normal time |
| Transfer | The same principle in a different surface: other dataset, other tool, other key, other client, other language | Solved, with the mapping stated explicitly |
| Retention | Re-run the transfer test cold after **≥30 days** | Same bar as transfer, no re-study first |
| Teaching | Explain to a novice, in writing or aloud, and answer their two hardest questions | The novice can act on it |

Two rules that make the ladder honest:

- **Promote one level at a time, dated, in `## Topics`.** A jump from Recall to Retention has skipped the two levels where most learning fails.
- **The Retention re-test is scheduled at pass time**, as a `## Due` row, not remembered. "Passed the transfer test" without a retention date is a promotion that gets silently reversed.

## Calibration

Track predicted versus actual. This turns confidence into a usable instrument instead of noise.

Protocol, every review and every test item:

1. Rate confidence 1-5 **before** checking the answer — recorded in the `Conf` column of the review queue.
2. Check.
3. The interesting cell is **high confidence + wrong**: a belief the learner will act on and never re-examine. These go to `## Error Log` marked as blind spots and get priority in the next drill.
4. Low confidence + right is the opposite problem: real knowledge treated as unreliable, which causes over-review and hesitation in performance. Reduce its review frequency deliberately.

A calibrated learner's 4s and 5s are right ~80-95% of the time. If they are right about half the time, no self-assessment from that learner is currently usable as evidence — verify externally until the gap closes.

## Signals of False Confidence

Observable, in order of how often they mislead:

| Signal | What it usually means |
|---|---|
| "That makes sense" / "I know this" during input | Fluency illusion: the explanation was easy to follow, which says nothing about producing it |
| Can complete it with the material visible, not without | Recognition, one level below what is being claimed |
| Correct only when the question is phrased as in the source | Memorised the surface, not the concept |
| Answers instantly on the familiar case, freezes on the variant | No model, one worked path |
| Can do it, cannot say why or when not to | Procedure without conditions — will misapply it confidently |
| Explains fluently in the source's exact phrasing | Repeating, not reconstructing — ask for a different analogy |
| Rates everything 5 | No calibration; the ratings carry no information |

The diagnostic that cuts through all of them: **"What is a case where this would be the wrong thing to do?"** Genuine understanding produces a boundary condition. Memorisation produces silence or a restatement.

## Understanding vs Memorisation

| Memorisation | Understanding |
|---|---|
| Exact phrasing survives, paraphrase breaks | Paraphrases freely, keeps the invariant |
| Fails on a rephrased question | Handles rephrasing and inversion |
| Cannot explain the cause | Explains why, and what would change the answer |
| No transfer to another surface | Applies to an unfamiliar instance |
| Cannot name a counter-case | Names the boundary and the exception |

Neither is bad. Memorisation is the correct target for arbitrary facts — vocabulary, keyboard shortcuts, dosages, notation. It is the wrong target for anything with a why. Deciding which one an item is happens at capture time (`capture.md`).

## The Cold Re-Test

The step almost universally skipped, and the reason "I learned this last year" is usually false.

- Schedule at pass time: a `## Due` row, ≥30 days out, or at the interval the item last survived (`schedule.md`).
- Cold means no re-reading, no warm-up, no notice the day before.
- Failure here is normal and cheap — information about the interval, not about the learner. Reset the topic to the level it actually holds and shorten the maintenance interval (`maintenance.md`).
- Passing a cold re-test is the only evidence that supports the word "learned" in `## Topics`.

## Failing Well

A failed verification is the most useful session in a topic — but only if it produces a diagnosis rather than a mood.

1. Classify the failure: **never encoded** (blank), **not retrievable** (tip-of-tongue, recognised on reveal), **misconceived** (confident and wrong), or **not assembled** (all pieces present, could not combine them).
2. Each maps to a different fix: encode → back to capture and the hard block; retrieve → shorten the interval; misconceive → contrast drill against the correct case; assemble → whole-task practice, not more drills (`practice.md`).
3. Write the classification in `## Error Log`. The class, not the score, is what makes the next attempt different.
4. Do not re-test the same day. A re-test after a fresh explanation measures the explanation.

## Certificates and Other Non-Evidence

| Thing | What it evidences |
|---|---|
| Course completion | Attendance |
| A green streak | Consistency, possibly of trivial sessions |
| Finishing a book | That the pages were turned |
| A passed multiple-choice exam | Recognition, plus test-taking skill |
| A tutorial project that runs | That the tutorial works |
| A blank-file rebuild of that project | Application |
| An unseen problem solved unaided, cold, a month later | Learned |

None of the first five is worthless — they are inputs. They are simply not the exit test, and treating them as the finish line is the most expensive substitution in self-directed learning; `plateaus.md` covers the collapse that follows discovering it.

Write the outcome the moment the test is graded: the level and its date into the topic's row in `## Topics`, the failure classification and any high-confidence miss into `## Error Log`, and the retention re-test into `## Due`. A test worth re-running — the task, its rubric, and each attempt's date and result appended — becomes `artifacts/assessment-<topic>-<stage>.md` with its `## Boxes` line, in the same turn. Formats in `memory-template.md`.
