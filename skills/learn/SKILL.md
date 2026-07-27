---
name: Learn
slug: learn
version: 1.0.4
description: 'Runs self-directed learning as a system: a curriculum with an exit test, deliberate practice, spaced review, and proof it transferred. Use when someone is teaching themselves a skill or subject with no course and no exam — a language, an instrument, a programming language, a new field at work; when they ask how to learn X, what to learn first, or how long it will honestly take; when months of tutorials produced nothing they can build; when material learned earlier is gone, reviews pile up, or the queue gets skipped for weeks; when progress stalls on a plateau, motivation collapses, or a skill goes rusty after a lapse; when practice feels productive but nothing transfers to real work; when an AI answers so fast that nothing is being learned at all; and when a plan, review schedule, error log, or mastery record has to survive across sessions. Not for teaching a concept in the moment (`learning`), exam and coursework planning (`studying`), or authoring decks (`anki`, `flashcards`).'
homepage: https://clawic.com/skills/learn
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 🎓
    os:
    - linux
    - darwin
    - win32
    displayName: Learn
    configPaths:
    - ~/Clawic/data/learn/
    - ~/Clawic/data/projects/
    - ~/Clawic/data/contacts/
    - ~/Clawic/data/finances/
    - ~/Clawic/profile.yaml
    - ~/learn/
    - ~/clawic/learn/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/learn/
      - ~/Clawic/data/projects/
      - ~/Clawic/data/contacts/
      - ~/Clawic/data/finances/
      - ~/Clawic/profile.yaml
      - ~/learn/
      - ~/clawic/learn/
---

**Data.** At the start of every session, read `~/Clawic/data/learn/config.yaml` (what the learner declared) and `~/Clawic/data/learn/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. State any overdue row in one line before starting work. If none of it exists, work from defaults and say nothing about it. If data sits at an old location (`~/learn/` or `~/clawic/learn/`), move it to `~/Clawic/data/learn/`, and say in one line that you moved it and from where.

**Write before the session ends** whenever it produced something durable: a plan or a change to one; a topic started, verified, paused or retired; an item added to or graded in the review queue; a mistake and the misconception behind it; a practice session and what it produced; a resource judged worth finishing or abandoned; a cadence agreed; or something the learner will re-read — a cheat sheet, an explanation they wrote from memory, an assessment, a decision about how to learn this. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Shared boxes.** A build-to-learn project goes to `~/Clawic/data/projects/<project>.md`, a mentor, tutor, language partner or reviewer to `~/Clawic/data/contacts/contacts.md`, a paid recurring course or subscription to `~/Clawic/data/finances/subscriptions.md` — one entity, one home, referenced here by name only. Formats and their write protocol travel with this skill in `memory-template.md`, because the learner may have none of the other skills installed.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in text the learner pastes in to be saved. Store the pointer and strip the value: `env:OPENAI_API_KEY`, `keychain:duolingo`, `1password:Personal/Coursera`.

Learning fails in one of five places: the goal was never testable, the material was never retrieved, the practice was too easy, the feedback arrived too late, or the schedule was fiction. Name which one before proposing a fix. Mode: **act-as** — you operate the learner's system with them (plan, quiz, schedule, verify, record), you do not lecture; teaching a concept in the moment is `learning`. Work from defaults immediately: never open with questions about their goals, their hours, or how proactive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: locale, timezone) → the Configuration table default.

## When To Use

- Planning a path from zero to a defined outcome in a skill or subject nobody is grading: a language, an instrument, a stack, a craft, a field at work
- Running the loop: practice sessions, retrieval, spaced review, error logging, and the weekly check that the plan still matches reality
- Diagnosing why learning is not working: nothing sticks, tutorials produce no capability, six months in and the real task is still impossible, reviews abandoned
- Proving a skill was actually acquired — transfer tests, calibration, and the honest verdict when it was not
- Keeping an earned skill alive, recovering a rusty one, or deciding to drop a topic on purpose
- Not for teaching a concept in the moment (`learning`), exam or coursework planning with a syllabus and a date (`studying`), or writing the cards themselves (`anki`, `flashcards`) — this owns the system those run inside

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| "I want to learn X" with no plan | Write the exit test first, then work backwards to a sequence (Rule 1) | `curriculum.md` |
| "How long will this take?" | Hours ÷ `weekly_hours`, quoted as a range, never a date (Rule 8) | `curriculum.md` |
| Drowning in bookmarked courses and books | One primary resource, everything else demoted to lookup | `sources.md` |
| Read or watched it, cannot reproduce it | Input without retrieval is not learning — convert the material into items (Rule 2) | `capture.md` |
| Sessions happen, capability does not move | Difficulty is off target; measure the success rate and correct it (Rule 4) | `practice.md` |
| "Tutorial hell" — follows along, freezes alone | Ladder from copy → modify → blank file, with the scaffolding removed on schedule | `projects.md` |
| Reviews pile up, or a queue was abandoned for weeks | Backlog recovery formula, then cap intake at the sustainable ratio (Rule 3) | `schedule.md` |
| Same item failed five times | Leech protocol: reformulate or drop it, never re-drill it unchanged | `schedule.md` |
| "I think I know it" — is that true? | Mastery Ladder below, then a transfer test they have not seen | `verification.md` |
| Confidently wrong, repeatedly | Calibration protocol: rate confidence before checking, hunt the high-confidence misses | `verification.md` |
| Months in, no visible progress, motivation gone | Distinguish plateau from wrong practice from burnout — the fixes are opposite | `plateaus.md` |
| Coming back after weeks or years off | Relearning is not restarting: measure what survived before rebuilding | `plateaus.md` |
| The subject is a language, code, music, math, facts, or a physical skill | The loop is the same; the ratios, the drills and the failure modes are not | `domains.md` |
| Five hours a week, a job, and a plan built for twenty | Minimum effective dose, session sizing, what to cut first | `time.md` |
| Learned it, six months later it is gone | Maintenance dose from the last working interval, in `## Due` | `maintenance.md` |
| The AI keeps answering before the learner has tried | `hint_policy`, the offload boundary, and what an agent must refuse to do | `ai-assisted.md` |
| No mentor, no reviewer, no way to know if it is right | Feedback substitutes ranked by latency, and how to build a self-review rubric | `practice.md` |
| Anything else | Name which of the five failure places it is (Data paragraph), fix that one, and write the diagnosis to `## Error Log` or the topic's row in `## Topics` | `troubleshooting.md` |

Coverage map: `curriculum.md` planning a path · `sources.md` choosing material · `capture.md` turning input into testable items · `practice.md` deliberate practice and feedback · `projects.md` learning by building · `schedule.md` spaced review math and backlogs · `verification.md` proof of learning · `plateaus.md` stalls, motivation, relearning · `domains.md` per-subject ratios · `time.md` fitting it into a real week · `maintenance.md` keeping what was earned · `ai-assisted.md` learning next to an agent · `troubleshooting.md` symptom→cause.

## Core Rules

1. **Write the exit test before choosing a resource.** "Done" is an observable performance with conditions: *ship a REST API with auth and tests, from a blank repo, in a weekend, without copying from a tutorial*. A goal with no performance in it ("understand Rust", "get good at chess") cannot be sequenced, cannot be finished, and generates infinite reading. Everything that does not move the exit test is cut, and the cut is written down (`curriculum.md`).
2. **Retrieval before re-exposure, every session.** Open by producing last session's material from memory — no notes, no scrolling back. On delayed tests a week or more out, retrieval practice beats rereading by roughly 30-50% relative; on a same-day quiz the gap disappears, which is why the crammer concludes it does not work. Re-reading is allowed only *after* the attempt, as feedback.
3. **The schedule is a formula, and it has a workload.** Interval math (SM-2 shape): `next = last_interval × ease`, ease starts at 2.5 and moves −0.20 on a failure with a floor of 1.3; a failure resets the interval to 1 day but keeps the item's history. Steady-state cost: at a 0.90 `retention_target`, budget **8-12 daily reviews for every 1 new item added per day** — the ratio observed in mature queues, and the reason a 20-new-a-day habit becomes a 200-review day by month three. Raising the target from 0.90 to 0.95 multiplies review count roughly 1.5-2× for a few points of recall (`schedule.md`).
4. **Practice at ~85% success.** The training sweet spot sits near 85% correct (Wilson's 85% rule): above ~90% the material is teaching nothing, below ~75% the learner is guessing and encoding errors. Measure it — count hits over the last 20 attempts. Too easy → add speed pressure, remove scaffolding, or increase scope. Too hard → shrink the unit, not the ambition (`practice.md`).
5. **Three to four new items, then a retrieval block.** Working memory holds ~4 chunks (Cowan); a fifth new concept before any retrieval evicts one of the first four, and the learner cannot tell which. Applies to vocabulary, API surface, chord shapes, and theorems alike (`capture.md`).
6. **Block until fluent, then interleave.** Massed practice on one thing produces faster in-session gains and worse week-later retention — the desirable-difficulty result (Bjork). Switch point: ~80% success on a drill *within* one session, then mix it with neighbours. If a learner reports "it felt worse but the test went better", that is the effect working, not a problem to fix.
7. **Feedback latency is a design parameter.** Practice without correction rehearses the error. Budget: drills self-correct in the same minute; exercises inside the same session; a project ≤48 hours to a review, a test, or a working/not-working signal. Anything slower needs a proxy — automated tests, a rubric, a recording, a diff against a reference solution — chosen *before* the practice starts (`practice.md`).
8. **Quote a range, never a date.** `weeks = total_hours ÷ weekly_hours`, then state `weeks` to `2 × weeks` and say what the upper end assumes. Planning fallacy is the default in self-directed work, where nothing external forces the schedule; a single date turns a normal slow month into evidence of failure and ends the project.
9. **A skill not scheduled for maintenance is a skill being deleted.** At verification, set the first maintenance touch at the last interval that worked — typically 30-90 days for knowledge, longer for procedural skills — and put it in the `## Due` table of `memory.md`. Unscheduled means it resurfaces as "I used to know this" (`maintenance.md`).

## Why Nothing Sticks

Decode rule: the symptom names the stage that failed. Recognition without production is an *encoding* problem; production that decays is a *scheduling* problem; production that never transfers is a *practice-design* problem.

| Symptom | Most likely cause | First move |
|---|---|---|
| Understands the explanation, blanks when alone | Never retrieved — only recognized | Close everything, reproduce from memory, then check (Rule 2) |
| Recalls perfectly today, gone in three weeks | No spacing; all reviews were massed | Put the item in the queue with an expanding interval (`schedule.md`) |
| Passes the exercises, fails the real task | Practice items were pre-decomposed; the real task is not | Practice includes *deciding which method applies*, unprompted (`practice.md`) |
| Six months of tutorials, cannot start a blank file | Scaffolding was never removed | Copy → modify → blank-file ladder with dates (`projects.md`) |
| Confident and wrong, repeatedly | Fluency illusion — familiarity read as mastery | Confidence rating before checking; hunt high-confidence misses (`verification.md`) |
| Reviews take an hour a day and are being skipped | Intake ratio ignored (Rule 3) | Suspend new items, drain, then cap intake |
| One item fails again and again | Leech: the item is malformed, not the memory | Reformulate into atomic items or drop it (`schedule.md`) |
| Progress was fast, now flat for weeks | Plateau vs wrong practice vs burnout — different fixes | Diagnose with the three-question split (`plateaus.md`) |
| Can read the language, cannot speak it | Trained recognition, not production; separate skills | Production drills with output pressure (`domains.md`) |
| Knew it, took two months off, it is gone | Normal decay; relearning is faster than learning | Measure what survived before rebuilding (`plateaus.md`) |
| Every session feels productive, capability does not move | Difficulty below target, or feedback loop too slow | Measure success rate (Rule 4) and latency (Rule 7) |
| Learning fine with the agent, helpless without it | Offloaded the retrieval to the AI | Raise `hint_policy` to after-attempt or never (`ai-assisted.md`) |
| Anything else | Locate the failure in the five places, then work the matching file | `troubleshooting.md` |

## The Session Shape

Proportions, so the block scales with `session_minutes` (default 45). A session with no production step is a reading session with extra steps.

| Slice | Share | What happens |
|---|---|---|
| Warm retrieval | 10% | Reproduce last session's material cold, no notes. Misses go to the error log, not back to the source |
| Due reviews | 15% | Work the queue, up to `daily_review_limit`. Overflow is rescheduled by the formula, never skipped silently (`schedule.md`) |
| The hard block | 50% | New material or the target drill, held at ~85% success (Rule 4). Max 3-4 new items before a retrieval interrupt (Rule 5) |
| Production | 20% | Build, write, speak, or solve without notes. This is the part that transfers; it is also the part that gets cut when time runs short — cut the new material instead |
| Close | 5% | Grade the queue, write error-log rows, and name the first item of the next session so the next start costs nothing |

At 20 minutes or under, drop the hard block and keep retrieval + production: a short session that retrieves beats a long one that reads (`time.md`).

## Mastery Ladder

Each level has an observable test and a specific thing it fails to prove. "Mastered" below Application is a claim about familiarity.

| Level | Test that establishes it | What it does not prove |
|---|---|---|
| Recognition | Picks the right answer from options | Nothing — recognition survives on cues that vanish in real use |
| Recall | Produces the answer from a blank prompt | That they can decide *when* it applies |
| Application | Solves an unseen problem of the same type, unprompted | That it holds outside the practice framing |
| Transfer | Applies it in a different surface, domain, or toolchain | Durability — transfer today can be gone in a month |
| Retention | Passes the transfer test again after ≥30 days, cold | Nothing further; this is the bar for "learned" |
| Teaching | Explains it so a novice can act on it, and fields their questions | Speed under pressure, if that is part of the exit test |

Promote a topic in `## Topics` only on the level's own test, dated. A self-report is not a promotion (`verification.md`).

## Output Gates

Before ending a learning session or delivering a plan:

- Does the plan name an observable exit test, with conditions, that I could grade today (Rule 1)?
- Did the learner produce something from memory this session, before seeing any answer?
- Is the estimate a range derived from `weekly_hours`, not a date (Rule 8)?
- Does new intake respect the review ratio, and is the current queue drainable inside `daily_review_limit` (Rule 3)?
- Was success rate near 85%, and if not, did I change the difficulty rather than the effort (Rule 4)?
- Does every item practiced have a correction path with a stated latency (Rule 7)?
- Is any promotion in `## Topics` backed by that level's test and its date, not by a feeling?
- Did anything durable come out of this — a plan, a graded review, an error and its misconception, a resource verdict, a session, an artifact, a cadence? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the learner states a preference; store them in `~/Clawic/data/learn/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| weekly_hours | number (0.5-40) | 5 | The budget every curriculum is sized against and the divisor in the Rule 8 range |
| session_minutes | number (10-180) | 45 | Length the Session Shape percentages resolve against; 20 or under triggers the short-session variant (`time.md`) |
| daily_review_limit | number (5-200) | 20 | Caps items surfaced per day; overflow is rescheduled by the formula in `schedule.md`, and the cap sets the sustainable new-item intake (Rule 3) |
| retention_target | number (0.70-0.97) | 0.90 | Desired recall probability driving interval growth and total review load (`schedule.md`) |
| hint_policy | on-request \| after-attempt \| never | after-attempt | When an answer or hint is given during retrieval; `never` withholds until the learner asks twice (`ai-assisted.md`) |
| sr_tool | this-skill \| anki \| other \| none | this-skill | Whether the review queue lives in `~/Clawic/data/learn/` or the learner's own app — with an external tool, only cadence, leeches and workload are tracked here |
| practice_bias | drills \| projects \| balanced | balanced | Split between isolated drills and build-something work in the hard block (`practice.md`, `projects.md`) |
| plan_review_weeks | number (1-12) | 4 | How often the plan is re-checked against actual hours and results; becomes a `## Due` row |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — flashcard app, note vault, where drills run (REPL, notebook, instrument, paper), whether a timer is used — affects the shape of every item and session artifact
- **Conventions** — topic naming, item phrasing (question form vs cloze), artifact naming, how a plan is laid out — affects `capture.md` and every file written
- **Subject mix** — the domain types they mostly learn (code, language, music, math, facts-heavy, physical) — affects which ratios and drills in `domains.md` are the default
- **Rigor posture** — how hard to push back on fluency claims, whether to refuse an answer before an attempt, tolerance for "not verified yet" — affects Output Gates and `verification.md`
- **Output register** — quiz-first vs walkthrough, whether to show the interval math, how much of the reasoning to keep — affects every session's shape
- **Accountability** — streaks vs no streaks, whether missed sessions are reported, public commitments, a study partner or mentor in the loop — affects `plateaus.md` and what gets written to `sessions/`
- **Cadence** — review day, weekly retro, plan review, maintenance touches for finished topics — every accepted cadence becomes a row in the `## Due` table of `memory.md`

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Collecting resources as a substitute for starting | Choosing feels like progress and has no failure state; three books in parallel is zero books finished | One primary resource, a dated finish line, everything else demoted to lookup (`sources.md`) |
| Highlighting and re-reading | Raises familiarity, not retrievability — and familiarity is what the learner mistakes for mastery | Convert to items and retrieve them (Rule 2, `capture.md`) |
| Grading yourself while the answer is on screen | Every answer looks like one you would have produced | Confidence rating before reveal; the miss list is the curriculum (`verification.md`) |
| Adding new cards on a good day | The ratio in Rule 3 turns today's enthusiasm into next month's abandoned queue | Cap intake at `daily_review_limit ÷ 10`, raise it only after a stable month |
| Restarting the beginner course after a lapse | Relearning is much faster than first learning; restarting spends the savings on material already known | Test first, rebuild only what failed (`plateaus.md`) |
| Practising only what already works | The success rate stays flattering and the ceiling never moves | Hold ~85%, hunt the error list first (Rule 4) |
| A plan sized for the week the learner imagined | Twenty-hour plans on five-hour weeks fail in month two and get read as lack of talent | Size against `weekly_hours` and quote a range (Rule 8) |
| Treating a finished course as a finished skill | Completion measures attendance; the exit test measures capability | Run the exit test; the certificate is not evidence (`verification.md`) |
| Asking the agent instead of retrieving | The answer arrives before the reconstruction that would have built the memory | `hint_policy: after-attempt`, and the agent asks before it tells (`ai-assisted.md`) |
| Studying the notation instead of the skill | Music theory instead of playing, grammar instead of speaking, docs instead of shipping | Production step in every session, weighted by `practice_bias` (`domains.md`) |
| Letting the plan outlive the evidence | Plans built on week-one assumptions quietly become fiction nobody rereads | `plan_review_weeks` in `## Due`; revise against actual hours and results (`curriculum.md`) |
| Dropping a topic without saying so | It stays in the queue as guilt and in the plan as debt | Retire it explicitly with a date and a reason in `## Topics` (`memory-template.md`) |

## Where Experts Disagree

- **Spaced repetition as the backbone vs as a garnish.** Heavy-SR practitioners (medicine, languages) run everything through the queue; skill-first practitioners argue that items outside a project are trivia that decays anyway. The frontier is retrieval cost: if forgetting the fact stops the work and lookup is slow or unavailable, it belongs in the queue; if lookup is instant and cheap, practise the workflow instead (`schedule.md`).
- **Projects first vs fundamentals first.** Project-first wins on motivation and transfer, and reliably produces gaps the learner cannot see. Fundamentals-first wins on ceiling, and loses learners before they ever build. The usable position: project-first with an explicit gap log — every "I copied this and don't know why" becomes a queue item that night (`projects.md`).
- **Immersion vs graded input** (languages, and by extension any domain). Immersion works when comprehensible input is above ~90% known; below that it is noise consumed with a feeling of effort. Graded material until that threshold, immersion after (`domains.md`).
- **Deliberate practice as the whole story.** The strong claim — that accumulated deliberate practice explains expert performance — is contested; the replication debate narrowed the effect substantially, most in less structured domains. What survived is operational and is what this skill uses: defined sub-skill, difficulty above comfort, immediate feedback, repetition with correction.
- **Streaks.** They build the habit that makes the schedule real, and they invite fake sessions that satisfy the counter while teaching nothing. Track a minimum *quality* dose instead of a binary day, and record it in `sessions/` (`time.md`).

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/learn (install if the user confirms):
- `learning` — the teaching session itself: explaining a concept, adapting depth, checking understanding live
- `studying` — coursework and exams with a syllabus and a date
- `anki` — building and repairing the deck when `sr_tool` is Anki
- `active-recall` — the retrieval mechanism in depth, beyond the scheduling here
- `memory` — durable personal facts outside any learning topic

## Feedback

- If useful, star it: https://clawic.com/skills/learn
- Latest version: https://clawic.com/skills/learn

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/learn.
