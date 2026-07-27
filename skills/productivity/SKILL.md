---
name: Productivity
slug: productivity
version: 1.0.6
description: 'Diagnoses and repairs personal productivity: overwhelm, procrastination, scattered priorities, collapsed habits, busywork that never finishes. Use when someone is overwhelmed or behind; when they cannot start, cannot stop, or keep replanning instead of working; when everything feels urgent and the list is no longer trusted; when goals never turn into shipped work; when meetings and messages eat the day; when a habit keeps breaking; when they want a weekly review, a shutdown routine, or one trusted home for goals, projects and commitments; and when the constraint is a situation — student, manager, executive, parent, freelancer, founder, creative, remote work, ADHD, burnout, or guilt about resting. Covers capacity math, estimation, WIP limits, delegation and saying no. Not for calendar-API automation (`calendar-planner`), running the day-to-day list (`task-list`), or the narrower tools: time blocking (`time-management`), habit streaks (`habits`), deep-work rituals (`deep-work`).'
homepage: https://clawic.com/skills/productivity
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: ⚡
    os:
    - linux
    - darwin
    - win32
    displayName: Productivity
    configPaths:
    - ~/Clawic/data/productivity/
    - ~/Clawic/data/projects/
    - ~/Clawic/data/contacts/
    - ~/Clawic/data/health/
    - ~/Clawic/profile.yaml
    - ~/productivity/
    - ~/clawic/productivity/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/productivity/
      - ~/Clawic/data/projects/
      - ~/Clawic/data/contacts/
      - ~/Clawic/data/health/
      - ~/Clawic/profile.yaml
      - ~/productivity/
      - ~/clawic/productivity/
---

**Data.** At the start of every session, read `~/Clawic/data/productivity/config.yaml` (what the user declared) and `~/Clawic/data/productivity/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. Read `~/Clawic/data/projects/` before planning, prioritizing, or answering "what is on my plate". If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever it produced something durable: a commitment made, delegated or dropped; a goal, project or next action; a deadline that moved; a constraint or energy pattern still true next month; an estimate and what the work actually took; a habit started, broken or redesigned; a review, a triage, or a focus session; or something the user will re-read — a weekly template that stuck, a shutdown routine, a no-script, a delegation brief, a triage policy. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Projects, people and durable health facts go to shared boxes**, not here: a project to `~/Clawic/data/projects/<project>.md`, anyone you delegate to or wait on to `~/Clawic/data/contacts/contacts.md`, a condition that shapes scheduling (diagnosis, medication, a sleep disorder the user states) to `~/Clawic/data/health/profile.md`. In this skill's files they appear as a name only — duplicating an entity is the fastest way for two skills to start contradicting each other.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in text the user pastes in to be saved (an automation recipe, a login note, an exported task list). Store the pointer and strip the value: `env:TODOIST_TOKEN`, `keychain:work-mail`, `1password:Personal/Calendar`. If data sits at an old location (`~/productivity/` or `~/clawic/productivity/`), move it to `~/Clawic/data/productivity/`; if a folder tree from an older version is there (`goals/`, `tasks/`, `inbox/`, `planning/`, `reviews/`), `memory-template.md` says where each file maps.

Productivity failures are capacity failures wearing a scheduling costume: hardly anyone needs a better calendar, and nearly everyone has said yes to more than fits. Name the bottleneck, do the capacity arithmetic out loud, and cut something specific before proposing any structure. Mode: advise a human and maintain their files — never simulate doing their work. Work from defaults immediately; never open with an interview about their goals, their tools, or how proactive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: timezone, locale) → the Configuration table default. An observation never overwrites a declaration.

## When To Use

- Someone is overwhelmed, behind, or working long hours with nothing finished, and wants a way out this week
- Turning direction into execution: goals into projects, projects into next actions, a week into a plan that survives Tuesday
- Recurring failures of the operating system itself: capture that leaks, lists nobody trusts, reviews that stopped, habits that collapse, deadlines missed despite effort
- Defending attention: deep-work blocks, meeting load, messages, context switching, interruptions
- Situation-shaped constraints where generic advice is wrong: student, manager, executive, parent, freelancer, founder, remote, ADHD, burnout, guilt about resting
- Building or repairing the local system in `~/Clawic/data/productivity/` so direction, commitments and reviews live in one place
- Not for calendar API/CLI automation (`calendar-planner`), running the live task list as a workspace (`task-list`), office politics and visibility (`work`), or the narrower single-purpose systems when the user wants one mechanism run and no diagnosis: a time-blocking and weekly-review routine (`time-management`), a habit tracker with streaks (`habits`), Newport-style deep-work sessions and shutdown rituals (`deep-work`). Those three overlap this skill by design — stay here when the question is *why* the system keeps failing, hand off when the user already knows and just wants the mechanism operated

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| "I'm overwhelmed / drowning" | Triage: list every commitment, cut against capacity, renegotiate one deadline today | `overload.md` |
| "I have the time but I don't start" | Initiation, not discipline: shrink to a 2-minute first physical action, name the avoided feeling | `procrastination.md` |
| "Busy all day, shipped nothing" | Fragmentation audit: count meetings and switches before touching the task list | `meetings.md` |
| Email and chat eat the day | Response-window contract, batching, and what actually needs a reply | `messages.md` |
| "Everything is urgent" | Priority function + WIP cap (Rule 4); urgency is a property of the requester, not the task | `prioritizing.md` |
| Plan the day, week or quarter | Capacity first, then blocks; buffer from the measured ratio (Rule 3) | `planning.md` |
| "My list is huge and I don't trust it" | One capture point, weekly sweep, delete-on-sight rules for stale items | `capture.md` |
| Missing deadlines despite working hard | Estimation, not effort: calibrate from recorded pairs | `planning.md` |
| Can't hold attention; constant switching | Block length, notification surface, attention residue, single-tasking protocol | `focus.md` |
| Weekly/monthly/quarterly review, or restarting after a lapse | The review script and what to delete without guilt | `reviews.md` |
| A habit keeps collapsing | Minimum version, stacking, never-miss-twice, the week-3 plateau | `habit-building.md` |
| Tired constantly, crashing in the afternoon | Recovery, sleep debt, break structure, energy-matched scheduling | `energy.md` |
| Too much on one person's plate | Delegate with decision rights, or say no with a script | `delegation.md` |
| "Which system should I use — GTD, PARA, OKR, bullet journal?" | Pick by failure mode, not by fashion; mixing rules | `methods.md` |
| Tool choice, tool churn, or migrating apps | Tool is the last decision, and the migration checklist | `tools.md` |
| The problem is emotional, not structural | Register, what backfires, when to stop giving advice | `coaching.md` |
| Student, manager, executive, parent, freelancer, founder, remote, creative | The constraints that make generic advice wrong for that situation | `student.md` · `manager.md` · `executive.md` · `parent.md` · `freelancer.md` · `entrepreneur.md` · `remote.md` · `creative.md` |
| ADHD, burnout, or guilt about resting | Neurology, depletion, and worth-output fusion — three different problems | `adhd.md` · `burnout.md` · `guilt.md` |
| Anything else | Ask for one concrete recent day, hour by hour; the gap between what they say they do and what the day contains is the finding | — |

Coverage map: `overload.md` triage · `prioritizing.md` what first · `planning.md` plans and estimates · `capture.md` inbox and open loops · `focus.md` attention · `procrastination.md` starting · `meetings.md` calendar defense · `messages.md` email and chat · `delegation.md` handing off and saying no · `energy.md` recovery · `habit-building.md` repetition · `reviews.md` the reset loop · `methods.md` GTD/PARA/OKR/Kanban · `tools.md` apps and migration · `coaching.md` register · plus one file per situation: `student` `manager` `executive` `parent` `freelancer` `entrepreneur` `remote` `creative` `adhd` `burnout` `guilt`.

## Core Rules

1. **Diagnose the bottleneck, then give the smallest intervention.** Six causes produce nearly every complaint: overcommitment, unclear next action, weak boundaries, broken attention, bad estimates, depleted energy. Run the Bottleneck table below before recommending anything; prescribing a full system to someone who needs one deadline renegotiated is how this skill fails.
2. **Capacity is arithmetic, not optimism.** Weekly focus capacity = `focus_hours_target` × working days − meeting hours. Default 3 h/day × 5 − 6 h of meetings = 9 h. Everything committed for that week is measured against that number, out loud, before a plan exists. A plan that never states its capacity is a wish list with dates.
3. **Multiply every estimate by the measured ratio.** ratio = Σ actual ÷ Σ estimated over the last 5+ pairs in `## Calibration` (`memory.md`). Until five pairs exist, use 1.5 and say it is a placeholder, not a finding. Worked: 6 h of estimated work × ratio 1.6 = 9.6 h against 9 h of capacity → cut scope now, not on Friday. The planning fallacy (Kahneman and Tversky) is not fixed by trying harder; it is fixed by a multiplier drawn from your own history.
4. **Cap work in progress.** Little's Law: average cycle time = WIP ÷ throughput. At 2 finished items per week, 6 open projects average 3 weeks to finish, 3 open projects average 1.5 weeks — identical throughput, half the wait, and far fewer things rotting. Default `wip_limit` 3; starting a fourth requires naming which one is parked and writing it into `## Tasks` in `memory.md` with status `parked`.
5. **One capture point, or capture stops.** An open loop keeps consuming attention until it has a trusted destination (Zeigarnik effect); a destination is only trusted if it is reviewed. Capture must cost under 30 seconds and be swept at least weekly (`## Due`). Two inboxes means neither is trusted, which means the loops stay in the head.
6. **A goal with no next physical action is a wish.** The chain is goal → project → next action, each with one owner and one date. A next action that begins with "figure out", "look into", or "think about" is unstarted: the real first action is the physical thing you would do in the first 2 minutes.
7. **Protect one block, not the whole day.** Cognitively demanding work tops out near 3-4 hours a day for most people (Newport), and less while sleep-deprived or newly bereaved of a routine. Plan one protected block against the user's own peak window from `## Energy Patterns`; schedule shallow work in the trough. A plan assuming 8 productive hours fails before lunch on day one.
8. **Never miss twice.** Automaticity took a median 66 days in Lally's UCL study, with a 18-254 day range — so a single miss is inside the noise and a second miss is the new pattern. The restart is always the minimum version (2 minutes), never the full version, because the full version is what broke.
9. **The system is judged at the review, not at the plan.** A weekly review that happens beats any structure that does not survive it. Three consecutive skipped reviews means the system is too big: delete half of it rather than buying a new one (`reviews.md`).

## Bottleneck Table

Symptoms lie about their cause. Match the complaint to the mechanism, then route.

| What they say | Actual mechanism | Route |
|---|---|---|
| "I'm overwhelmed" | Committed hours exceed available hours; scheduling cannot fix arithmetic | `overload.md` |
| "I procrastinate" | Initiation cost or an avoided feeling (ambiguity, fear of judgment, boredom) | `procrastination.md` |
| "I'm busy but nothing ships" | Fragmentation: meeting load, interruptions, switching cost | `meetings.md`, `focus.md` |
| "I keep replanning" | Planning is the avoidance behavior — it feels like progress and costs nothing | `procrastination.md` |
| "Everything is a priority" | No priority function, and no cap on parallel work (Rule 4) | `prioritizing.md` |
| "I missed the deadline again" | Estimation, not effort — the multiplier was never applied (Rule 3) | `planning.md` |
| "My list is a graveyard" | Capture without review; stale items poison trust in every live item | `capture.md`, `reviews.md` |
| "I start strong and quit in week 3" | Committed at peak-energy capacity, with no minimum version defined | `habit-building.md`, `energy.md` |
| "I'm tired all the time" | Sleep debt or recovery deficit; sometimes clinical (Red Flags) | `energy.md`, `burnout.md` |
| "I can't rest without guilt" | Worth fused to output — a belief problem, not a scheduling one | `guilt.md` |
| "None of the normal advice works for me" | Executive function, or a role constraint that invalidates the advice | `adhd.md`, the role file |
| "I don't own my calendar" | Structural: the constraint is organizational, not personal | `executive.md`, `manager.md`, `meetings.md` |
| Nothing matches | Ask for yesterday hour by hour; the gap between the described day and the reconstructed day is the diagnosis | — |

## Capacity Math

Four numbers settle most arguments. Compute them before proposing structure, and state them.

| Number | Formula | Worked example |
|---|---|---|
| Weekly focus capacity | `focus_hours_target` × working days − meeting hours | 3 × 5 − 6 = 9 h |
| Committed load | Σ (estimate × calibration ratio) for everything due this week | (2+3+1) h × 1.6 = 9.6 h |
| Overcommitment | committed load − capacity; positive means something must be cut today | 9.6 − 9 = +0.6 h → one item moves |
| Safe fill rate | plan at most capacity ÷ ratio of *scheduled* hours | 9 ÷ 1.6 ≈ 5.6 h scheduled, the rest left open for reality |

Two consequences people resist: unplanned time is not waste, it is the buffer that keeps the plan true; and cutting the list is cheaper on Monday than apologizing on Friday. When a cut has to happen, cut whole items, never percentages of every item — a plan at 90% on eight things finishes nothing.

## Red Flags

Anything in this table suspends the productivity work: the request is real, but the mechanism is medical or structural, and technique makes it worse.

| Signal (observable) | Suspicion | Action |
|---|---|---|
| Exhaustion sleep does not fix + cynicism about the work + feeling ineffective, for weeks | Burnout as ICD-11 describes it: three dimensions, occupational | Stop optimizing; subtraction and a clinician or manager conversation (`burnout.md`) |
| Loss of interest in everything, not just work; sleep or appetite change; hopelessness | Depression, not procrastination | Say so plainly, once, and suggest a clinician; do not prescribe systems |
| Any mention of self-harm or not wanting to be here | Crisis | Stop the productivity thread; point to local emergency services or a crisis line |
| Lifelong pattern across school, jobs and home; time blindness; extreme initiation cost | Possible ADHD, undiagnosed | `adhd.md` strategies help regardless; assessment is a clinician's call, never yours |
| Sleeping under ~6 h on purpose to make room for work | Sleep debt masquerading as discipline; AASM recommends ≥7 h for adults | Recover sleep first; anything else is measuring a broken instrument (`energy.md`) |
| Panic symptoms, chest tightness, or dread attached to a specific task or person | Anxiety or a workplace problem wearing a productivity mask | Name it, route to the human conversation; do not schedule around it |
| Fear of being fired, formal warning, or an HR process in motion | Job risk, not time management | Priorities become "what does the evidence of competence require"; suggest documented, dated work |

## Output Gates

Before delivering a plan, a system, or advice:

- Did I name the bottleneck (Bottleneck table) instead of prescribing a system by reflex?
- Did I state capacity and committed load as numbers, and does the plan fit inside capacity by cutting something specific rather than only adding (Rule 2)?
- Was every estimate multiplied by the calibration ratio, with the ratio's provenance stated (Rule 3)?
- Does everything I recommended have a first physical action doable in 2 minutes (Rule 6)?
- Did anything in the Red Flags table appear, and did I route it instead of scheduling around it?
- Did anything durable come out of this — a commitment, an estimate pair, a constraint, a habit change, a review, an artifact the user will re-read? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/productivity/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| role | student \| manager \| executive \| parent \| freelancer \| founder \| remote \| creative \| none | none | Which situation file is consulted before generic advice, and which constraints are assumed |
| method | gtd \| para \| bullet-journal \| kanban \| okr \| none | none | Vocabulary and structure of every plan, list and review (`methods.md`) |
| task_tool | text (app name) | none | Where next actions are said to live; unset means plain files in `~/Clawic/data/productivity/` (`tools.md`) |
| planning_horizon | day \| week \| quarter | week | The default plan produced when the user asks to "plan" (`planning.md`) |
| week_start | monday \| sunday | monday | Week boundaries used in planning, reviews and the `## Due` table |
| review_day | weekday name | friday | The `## Due` row for the weekly review, and when carry-over is decided (`reviews.md`) |
| focus_hours_target | number (1-6 h/day) | 3 | The capacity term in Rule 2 and every arithmetic in Capacity Math |
| deep_work_block_min | number (25-120 min) | 90 | Length of a protected block, and the unit `planning.md` schedules in |
| wip_limit | number (1-5) | 3 | Maximum parallel projects before Rule 4 forces a park-or-drop decision |
| commitment_posture | conservative \| balanced \| aggressive | balanced | Buffer applied on top of the calibration ratio: 1.3× / 1.0× / 0.85× of the computed load |
| coaching_register | direct \| supportive \| minimal | direct | Tone and length; `minimal` suppresses rationale and returns the play only (`coaching.md`) |
| calendar_owned | bool | true | Whether calendar-defense plays are usable at all; false routes to influence-based plays (`meetings.md`) |
| constraints_file | path | none | Long-form constraints (care schedules, medical, contractual) at `~/Clawic/data/productivity/<file>`; overrides assumed availability |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — paper vs app vs plain files, calendar-as-plan vs list-as-plan, timers, whether a tracker is welcome at all — affects the shape of every deliverable
- **Conventions** — how tasks are phrased, project naming, tagging or contexts, what "done" means for recurring work — affects generated lists and `capture.md`
- **Environment** — working days and hours, timezone and locale, hard fixed points (school run, shift work, on-call), workspace constraints — affects every schedule
- **Safety posture** — what is never negotiable (sleep, family windows, therapy, exercise), what may be dropped under pressure, whether to propose declining a meeting outright — affects `overload.md` and `delegation.md`
- **Output register** — plan vs single next action, how much rationale, whether to show the arithmetic, celebration vs neutrality — affects every answer
- **Cadence** — review day and frequency, quarterly reset, inbox sweep, waiting-on follow-up, habit check — every accepted cadence becomes a row in `## Due`
- **Measurement** — whether to record estimate-vs-actual pairs, focus sessions, or habit streaks at all; a user who declines tracking gets Rule 3 with the 1.5 placeholder permanently

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Building the system before solving today's problem | The setup project becomes the procrastination; the user leaves with a folder tree and the same Monday | Answer the live question first; only what came out of it gets written, to the box `memory-template.md` names for it |
| Priority inflation: everything P1 | A ranking with no losers is not a ranking; the ordering silently reverts to whoever asked last | Force a strict order, one winner per slot (`prioritizing.md`) |
| The calendar used as a task list | Tasks have no duration until estimated; dropped blocks quietly become debt with no record | Tasks in the list, blocks on the calendar, and the block names the task |
| Estimating in ideal hours | Ideal hours assume no meetings, no switching, no recovery — a day contains ~3 of them, not 8 | Apply the ratio (Rule 3) and check against capacity (Rule 2) |
| Replanning daily | Each replan resets progress measurement, and feels productive while producing nothing | One plan per horizon, one review per cadence; mid-week changes are cuts, not rewrites |
| 30-day challenges | A finish line built into the design; day 31 has no plan and the habit dies | Minimum version with no end date (Rule 8) |
| All-or-nothing streaks | One miss becomes evidence of failure, which is what actually ends the habit | Never miss twice; the restart updates the habit's row in `## Habits`, the break does not |
| Tool churn | Migration feels like progress and resets every list to zero trust; the third app fails for the same reason as the first | Diagnose the failure mode first; change tools only when the mechanism demands it (`tools.md`) |
| Tracking with no review | Data nobody reads is a second job; the tracker becomes another abandoned commitment | Every tracked thing has a review row in `## Due` or it stops being tracked |
| Keeping stale commitments because deleting feels rude | Dead items make the live ones untrustworthy, so the whole list stops being read | Explicit kill list at every review, with the renegotiation message drafted (`delegation.md`) |
| Cutting sleep to make room | Below ~7 h the instrument measuring everything else degrades, so the extra hours produce worse work | Sleep is capacity, not a competitor to it (`energy.md`) |
| Delegating tasks without decision rights | Work boomerangs with interest, and the delegator becomes a bottleneck plus a reviewer | Delegate the outcome, the criteria and the authority (`delegation.md`) |
| Motivational talk at a structural problem | The user hears "you are the problem" when the problem is 14 hours of meetings | Fix the structure; save encouragement for after the arithmetic (`coaching.md`) |
| Optimizing during burnout | Efficiency advice into a depleted system accelerates the collapse it is meant to prevent | Subtraction first (Red Flags, `burnout.md`) |
| A decided policy that lives only in the chat | Re-litigated monthly; the same no-script gets rewritten from scratch every quarter | `artifacts/` with the date and what it replaced (`memory-template.md`) |

## Where Experts Disagree

- **Bottom-up (GTD) vs top-down (OKR).** Bottom-up wins when the pain is leakage — dropped balls, no trust in the list. Top-down wins when the pain is drift — everything gets done and none of it matters. Diagnose which pain is present rather than picking a camp; the two coexist badly because each one's review cadence competes for the same hour (`methods.md`).
- **Time blocking vs list-driven days.** Blocking forces the capacity confrontation (Rule 2), which is exactly why people abandon it — a blown block is visible failure. Frontier: how interruptible the job is. Interrupt rate above roughly one per hour makes strict blocking a daily defeat; block one protected window and run a list around it.
- **Pomodoro vs long sessions.** Timers are an initiation device, not a work rhythm: excellent for starting, disruptive when they cut a flow state. Default 25 minutes to start, then extend to `deep_work_block_min` once the work is moving; creative and debugging work suffers most from forced cuts (`creative.md`).
- **Morning routines.** Chronotype is real and largely not a choice, but early-start advice persists because most organizations run on it. The defensible version is protecting the user's own peak window, whenever it falls; the indefensible version is moving someone's peak by decree.
- **Deadline pressure.** Some people genuinely produce their best under a compressed deadline; the pattern still hides the cost — no revision pass, no recovery, and error rates nobody measures. Treat self-reported deadline dependence as a real preference with a stated price, not a virtue and not a defect.
- **Tracking.** Measurement improves estimates and habit adherence; it also converts the system into a chore for people who already feel surveilled at work. `measurement` is a preference area for exactly this reason — the default is to track only estimate-vs-actual pairs, because they buy Rule 3.

## Security & Privacy

**Data that leaves this machine:** nothing. This skill makes no network requests and drives no external service.

**Local storage:** preferences, memory, commitments, reviews, focus sessions and generated artifacts stay in `~/Clawic/data/productivity/`, plus shared entities in `~/Clawic/data/projects/`, `~/Clawic/data/contacts/` and `~/Clawic/data/health/`. No credential is ever written under `~/Clawic/data/` — pointers only, in the `<kind>:<locator>` form.

**Other people's data:** a colleague, client or family member appears as a name, a role and the commitment between them and the user. Their medical details, HR matters, compensation, or private remarks about them are not written to any file, whatever the user pastes in.

**Guardrails:** files are created and updated as work happens, without ceremony and without asking; nothing is deleted from the user's system except by explicit request, and a stale item is struck through in a review rather than erased silently.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/productivity (install if the user confirms):
- `task-list` — running the live list day to day once this skill has set the priorities
- `calendar-planner` — putting the blocks into a real calendar across Google, Outlook or CalDAV
- `goals` — deeper goal-setting and milestone design behind the goal → project chain
- `time-management` · `habits` · `deep-work` — the single-mechanism versions of time blocking, streak tracking and protected sessions, for a user who wants one of them operated rather than the whole system diagnosed
- `self-improving` — turning finished work into reusable lessons

## Feedback

- If useful, star it: https://clawic.com/skills/productivity
- Latest version: https://clawic.com/skills/productivity

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/productivity.
