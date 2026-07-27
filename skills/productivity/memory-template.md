# Working File Templates — Productivity

Read this file only when WRITING. `config.yaml` is what the user **declared**; `memory.md` and everything it indexes is what you **observed** or produced together. An observation never overwrites a declaration.

## Where each thing goes

| Data | Home | How it grows |
|---|---|---|
| Declared preferences — table keys and preference areas alike | `~/Clawic/data/productivity/config.yaml` | Key by key, read-modify-write |
| Constraints, energy patterns, working style, friction, goals, live tasks, commitments, habits, estimate calibration, parked ideas, due dates, box index | `~/Clawic/data/productivity/memory.md` | Rewritten in place; stays small |
| Long-form constraints the user wrote out (care schedule, contract terms, medical restrictions) | `~/Clawic/data/productivity/<name>.md`, path stored in `constraints_file` | One file, replaced when the user revises it |
| A project: goal, status, milestones, decisions | `~/Clawic/data/projects/<project>.md` (**shared**) | One file per project, from the first |
| A person delegated to, waited on, or promised something | `~/Clawic/data/contacts/contacts.md` (**shared**) | One row per person; the commitment itself stays in `## Commitments` |
| A durable health fact the user states that shapes scheduling (diagnosis, medication, sleep disorder) | `~/Clawic/data/health/profile.md` (**shared**) | One dated line per fact |
| A weekly, monthly or quarterly review | `~/Clawic/data/productivity/reviews/<year>.md` | Append-only, cut by year |
| A focus session, time audit, or estimate-vs-actual pair worth keeping in detail | `~/Clawic/data/productivity/sessions/<year>.md` | Append-only, cut by year; the derived ratio lives in `## Calibration` |
| Things you produced that get re-read — a weekly template that stuck, a shutdown routine, a no-script, a delegation brief, a triage policy, a quarterly goal set, a meeting charter, a role scorecard, an operating manual | `~/Clawic/data/productivity/artifacts/<kebab-name>.md` | Born as its own file, from the first one |
| **Anything durable this table does not name** | `~/Clawic/data/productivity/<plural-noun>.md`, or `artifacts/<kebab-name>.md` if it is a long text read whole | Name the file after what it holds, never after when it was made — the one exception is a series that recurs (an audit, an export, a quarterly set), where the noun still comes first and an ISO date is only a suffix: `meeting-audit-2026-07.md`, never `2026-07-notes.md` (the `<date>` in every path in this skill means exactly that suffix); add its `## Boxes` line in the same turn |
| Credentials of any kind | Nowhere under `~/Clawic/data/` | Pointer only — see Secrets |

Deciding where something unnamed goes, in order: (1) would another skill want to read it? → the shared box it belongs to, below. (2) Is it a text read whole when its subject comes up — a procedure, a policy, a decision with its reasoning, a template? → `artifacts/`, its own file from the first one. (3) Is it one more row of something that accumulates? → a section of `memory.md` until the split threshold, then its own box.

## When to write

No permission needed; every write is announced in one line that names the file. Writes and deletions stay inside the paths declared in this skill's `configPaths`. A deletion is named in that same line, and in a shared box only rows this skill itself wrote are ever updated or removed.

| It happened | Write |
|---|---|
| A commitment was made, renegotiated, delegated or dropped | `## Commitments`, and the person's row in `contacts.md` |
| A task was captured, started, finished or killed | `## Tasks` |
| A goal was set, revised, hit or abandoned | `## Goals` |
| Work was scoped as a project | `~/Clawic/data/projects/<project>.md`, name only in `## Goals` |
| An estimate was made and the work finished | The pair in `## Calibration`, detail in `sessions/<year>.md` |
| A weekly, monthly or quarterly review ran | `reviews/<year>.md`, and the `## Due` row |
| A focus session or time audit produced numbers | `sessions/<year>.md` |
| A habit started, broke, restarted or was redesigned | `## Habits` |
| A fixed constraint appeared (school run, on-call, therapy, shift) | `## Constraints` |
| An energy pattern showed up twice | `## Energy Patterns` |
| A recurring structural problem was named | `## Friction` |
| A template, script, routine, policy or goal set came out of the session | `artifacts/` |
| An idea was parked without commitment | `## Someday` |
| A cadence was agreed or run | `## Due` |
| The user declared a preference | Its key in `config.yaml` |
| The user stated a durable health fact that changes scheduling | `~/Clawic/data/health/profile.md` |

Nothing is recorded because it was said once in passing: a constraint is written when the user states it as ongoing or you have seen it twice, and a preference only when the user declares it.

## Start flat, split only when it hurts

Everything except reviews, sessions, artifacts and the shared boxes begins inside `memory.md`. Splitting is a procedure, not a suggestion:

1. Before appending to a section, count its entries.
2. If the append would take it past **~15 entries or ~40 lines of real content** — scaffolding, headings and comments do not count — then, in the same turn: create the new file in `~/Clawic/data/productivity/`, move the whole section into it, **delete the section from `memory.md`**, add its line to `## Boxes`, and append the new entry to the new file.
3. Keep the headings identical on both sides of the move, so the split is a copy-paste and never a rewrite.
4. Never leave a copy behind. If the same data ever appears in both places, the extracted file wins and the `memory.md` copy is deleted.

The sections that have a split target, with the file they become: `## Tasks` → `tasks.md` · `## Commitments` → `commitments.md` · `## Goals` → `goals.md` · `## Habits` → `habits.md` · `## Someday` → `someday.md`. `## Calibration` never splits: it is capped at the last 10 pairs, oldest dropped, because a two-year-old ratio describes a person who no longer exists.

Reviews, focus sessions and artifacts are the exception: each is born in its own box whatever its size, because a review is read by date and an artifact is read whole, only when its subject comes up.

## Secrets

Nothing under `~/Clawic/data/` ever holds a secret value — not the files named here, not files you create, not text the user pastes in and asks you to keep. Store the pointer in its place, in this shape: `<kind>:<locator>`.

`env:TODOIST_TOKEN` · `keychain:work-mail` · `1password:Personal/Calendar` · `bitwarden:Work/VPN` · `file:~/.ssh/id_ed25519` · `profile:work`

When the user pastes something to save — an automation recipe, an exported list, a login note, a meeting dial-in — replace each secret value before writing and leave the pointer visible: `api_token: <env:TODOIST_TOKEN>`. Say in one line that you did it.

In this domain — **not secrets, keep them**: task and project names, goal statements, deadlines, first names and roles, employer and client names, tool and app names, meeting titles, habit names, review notes, focus-session numbers, energy windows. **Secrets, strip them**: API tokens for task or calendar apps, email and account passwords, 2FA and recovery codes, meeting-room PINs and dial-in passcodes, VPN or SSO credentials, anything the user labels private access.

Separate from secrets, and just as firm: **other people's private information is not written here.** A colleague, client or family member appears as a name, a role, and the commitment between them and the user. Their medical details, HR matters, pay, or the user's private opinion of them do not go into any file, however the user phrases the request.

**Contents:** [config.yaml](#configyaml) · [memory.md](#memorymd) · [shared boxes](#shared-boxes) · [reviews/](#reviews) · [sessions/](#sessions) · [artifacts/](#artifacts) · [split-out files](#split-out-files) · [legacy layout](#legacy-layout)

## config.yaml

Keys come from the Configuration table in `SKILL.md`, plus free-form keys nested under a preference area. Write a key only when the user states the preference.

**Writing is read-modify-write**: load the existing file, set or replace only the key just declared, keep every other key byte for byte. Never emit a `config.yaml` from this template — the template shows shape, not content. Create `~/Clawic/data/productivity/` if it does not exist.

```yaml
role: manager
method: gtd
task_tool: Things
planning_horizon: week
week_start: monday
review_day: friday
focus_hours_target: 3
deep_work_block_min: 90
wip_limit: 3
commitment_posture: conservative
coaching_register: direct
calendar_owned: false
constraints_file: constraints.md

# Preference areas — free-form keys added as the user reveals them.
# A preference the user states is a declaration and belongs here, never in memory.md.
conventions:
  task_phrasing: "verb + object + where"
safety_posture:
  never_negotiable: [sleep, "school run 15:00", "therapy tuesday"]
output_register:
  show_arithmetic: false
measurement:
  track: [estimate_pairs]      # declined habit streaks and session logs
cadence:
  quarterly_reset: first week of the quarter
```

If you find a preference recorded in `memory.md`, move it here and note the move.

## memory.md

Write only the sections you have content for — a heading with nothing under it is noise, and it inflates the line count that decides a split. Never copy these hints into the user's file. `## Boxes` is the one section that is never dropped when `memory.md` is rewritten: deleting a line there orphans a file forever. This is what a populated file looks like:

```markdown
# Productivity Memory

## Status
status: ongoing
last: 2026-07-26

## Boxes
- Tasks (23 open) → `tasks.md`; read before any planning, prioritizing, or "what should I do now"
- Reviews 2026 (18) → `reviews/2026.md`; read the last two before running a review or a quarterly reset
- Focus sessions 2026 (61) → `sessions/2026.md`; read before changing block length or re-deriving the ratio
- Shutdown routine → `artifacts/shutdown-routine.md`; read when the day will not end or work bleeds into the evening
- No-scripts for scope creep → `artifacts/no-scripts.md`; read before declining, renegotiating, or pushing back

## Due
| What | Every | Last run | Next due |
|------|-------|----------|----------|
| Weekly review | week, friday | 2026-07-24 | 2026-07-31 |
| Inbox sweep | week, friday | 2026-07-24 | 2026-07-31 |
| Waiting-on nudge | week, monday | 2026-07-20 | 2026-07-27 |
| Monthly review | month, last friday | 2026-06-26 | 2026-07-31 |
| Quarterly goal reset | quarter | 2026-07-02 | 2026-10-01 |
| Habit check | month | 2026-07-01 | 2026-08-01 |

## Constraints
School run 15:00-16:00 daily. On-call every third week, no deep work. Team standup 09:30 Mon-Thu.

## Energy Patterns
Peak 07:00-10:00, hard trough 14:00-15:30. Two 90-minute blocks is the ceiling; a third produces work that gets rewritten.

## How They Work
Wants the play, not the theory. Responds to arithmetic, resists encouragement. Plans well, starts badly.

## Friction
Says yes in the meeting, regrets it the same afternoon — the cost is never computed live. Four projects open against a wip_limit of 3 since June.

## Goals
| Goal | Why | Deadline | Project | Status |
|------|-----|----------|---------|--------|
| Ship the billing rewrite | unblocks the pricing change | 2026-09-30 | billing-rewrite (projects box) | on track |

## Tasks
| Task | Project | Due | Est h | Status |
|------|---------|-----|-------|--------|
| Draft migration plan | billing-rewrite | 2026-07-29 | 2 | doing |
| Review Ana's proposal | — | 2026-07-28 | 1 | next |

## Commitments
| Direction | What | Who | Due | Last nudge |
|-----------|------|-----|-----|------------|
| owed by me | pricing memo | Ana (contacts) | 2026-07-30 | — |
| owed to me | infra estimate | Marco (contacts) | 2026-07-25 | 2026-07-24 |

## Habits
| Habit | Minimum version | Trigger | Started | Last break | State |
|-------|-----------------|---------|---------|------------|-------|
| Morning planning | read yesterday's plan | after coffee | 2026-05-04 | 2026-07-19 | recovered next day |

## Calibration
ratio 1.6 (8 pairs, last 90 days)

| Work type | Estimated h | Actual h |
|-----------|-------------|----------|
| Writing (memo) | 2 | 3.5 |
| Code review | 1 | 1.2 |

## Someday
Learn the reporting stack. Rewrite the onboarding doc. Conference talk in spring.

---
*Updated: 2026-07-26*
```

Rules that keep this readable next month:

- **`## Boxes`**: one line per file that exists — `<what> (<volume>) → <file>; read when <condition>`. Written in the same turn the file is created. Never delete a line without deleting the file it points to. A box with no index line does not exist.
- **`## Due`**: check it against today's date at the start of a session and state any overdue item in one line — a statement, not a question. Every recurring thing this skill schedules belongs here, and nothing is scheduled that has no row.
- **`## Tasks`**: live items only. A finished task is deleted, not archived with a strike-through — the record that matters is the review. `Est h` is the raw estimate, before the ratio, so the pair stays comparable.
- **`## Commitments`**: `Direction` is `owed by me` or `owed to me`; the person is a name pointing at `contacts.md`, never a duplicated contact record. A dropped commitment is deleted here and named in the next review, so "what did I drop this quarter" has an answer.
- **`## Calibration`**: ratio = Σ actual ÷ Σ estimated across the listed pairs. Keep the last 10 pairs, drop the oldest, and recompute the ratio in the same turn a pair is added — SKILL.md Rule 3 reads this number, and a stale ratio silently corrupts every plan.
- **`## Goals`**: one row per goal. The `Project` column holds the project name only; the project's own state lives in the shared projects box.
- These headings are exactly the ones the split-out files get, so a split is a copy-paste.

| Status | Meaning |
|-------|---------|
| `ongoing` | Still learning how they work |
| `complete` | Constraints, energy pattern and working style are known and stable |

## Shared boxes

Three boxes are shared with every other Clawic skill. The user may have none of the owning skills installed, so the format and the protocol travel with this one. Read the file before adding and look for the identity key: if it is there, **update in place**; only its absence justifies a new row. Update and retire your own entries, never another source's. If the file already exists with different columns, **match its columns** and add anything missing as a trailing note — never rewrite its header. Amounts and measures carry their unit inside the value (`3 h`, `62 USD`), because these boxes get summed by skills that cannot guess yours.

**Projects** — `~/Clawic/data/projects/<project>.md`, one file per project from the first; identity key = the project name (the file slug).

```markdown
# Billing rewrite

status: active            # active | blocked | done | cancelled — with the date on close
goal: replace invoicing so pricing can change without a deploy
owner: user
deadline: 2026-09-30
milestones:
- [x] schema agreed — 2026-07-10
- [ ] migration plan — 2026-07-29
decisions:
- 2026-07-10: keep legacy IDs; renumbering breaks every historical invoice
```

Closing is `status: done | cancelled — <date>` inside the file, never deletion: the file is the record of what was delivered. Past ~20 closed projects, move them to `projects/archive/<project>.md` without renaming.

**Contacts** — `~/Clawic/data/contacts/contacts.md`, one table; identity key = `Key` (lowercase email → handle → `<kebab-name>` plus a stable disambiguator).

```markdown
| Name | Key | Role | Preferred channel | Context | Last contact | File |
|------|-----|------|-------------------|---------|--------------|------|
| Ana Ruiz | ana@acme.com | client, Acme | email | owns the pricing decision | 2026-07-24 | — |
```

`Preferred channel` is the channel type, not the address, and never the identity key. Past 15 people, or as soon as one no longer fits in a row, split to `~/Clawic/data/contacts/<name>.md` per person and leave `contacts.md` as the index with the `File` pointer. Someone who is no longer a working relationship is retired by deleting the row and noting the date in `memory.md`. Write only what the working relationship needs.

**Health** — `~/Clawic/data/health/profile.md`; identity key = the fact plus its date. Write here only what the user states about themselves and what changes scheduling: a diagnosis they name, medication whose timing shapes the day, a sleep or pain condition. One dated line each, under the heading the file already uses (`## Conditions`, `## Medication`). Never a symptom you inferred, never a third party's health, never a diagnosis of your own — SKILL.md Red Flags route that to a clinician instead. Retiring a fact means replacing the line and dating the change, because "resolved in 2026-03" is itself information.

## reviews/

```markdown
# Reviews — 2026

## 2026-07-24 — weekly
Shipped: migration plan draft, pricing memo.
Stalled: onboarding doc, third week running — kill or schedule.
Dropped: conference CFP. Told Ana on 07-24.
Capacity: 9 h planned, 11.5 h actual; ratio moved 1.5 → 1.6.
Next week's one priority: migration plan signed off.

## 2026-06-26 — monthly
Pattern: every overrun started as a yes given inside a meeting.
Change for July: no same-day yes to anything above 2 h.
```

Weekly entries stay short enough that a year of them reads in one pass. A monthly entry names one pattern and one change. A quarterly entry closes or renews each goal explicitly, so no goal survives by inertia.

## sessions/

```markdown
# Focus sessions — 2026

| Date | Work | Planned min | Actual min | Interruptions | Estimated h | Actual h |
|------|------|-------------|------------|---------------|-------------|----------|
| 2026-07-24 | migration plan | 90 | 55 | 3 (chat, standup, door) | 2 | 3.5 |
```

This box exists to buy two numbers: the block length the user actually sustains, and the calibration ratio. Add the estimate pair to `## Calibration` in the same turn and recompute the ratio there. If the user declined measurement (`measurement` preference area), this box is never created and Rule 3 keeps the 1.5 placeholder permanently.

## artifacts/

One file per thing, at `~/Clawic/data/productivity/artifacts/<kebab-name>.md`, created the first time it exists. Canonical types here: **a planning or review template that stuck**, **a shutdown or startup routine**, **no-scripts and renegotiation messages**, **a delegation brief**, **a triage policy**, **a quarterly goal set**, **a meeting charter**, **an operating manual** ("how I work", for a new manager or client). Every artifact opens with when to read it, and gets its `## Boxes` line in the same turn.

```markdown
# Shutdown routine
*Read when: the day will not end, or work bleeds into the evening. Written 2026-07-26.*

1. Tomorrow's one must-win, written before anything is closed.
...
```

```markdown
# No-scripts — scope creep
*Read before declining, renegotiating, or pushing back on a request. 2026-07-26.*

Trade: "Happy to add that — it moves delivery to <date>, or it replaces <item>. Which?"
Decline: "I can't take that on and give it the attention it needs."
Rejected: apologising first — it turns the negotiation into one about the apology instead of the scope.
```

If the artifact belongs to work the user tracks as a project, its decision summary also belongs in `~/Clawic/data/projects/<project>.md`, with the full text staying here and referenced by name.

## Split-out files

Created only by the split procedure above, never on day one. Each keeps the exact headings it had inside `memory.md`.

`tasks.md` — `## Tasks`, same columns. The first box most users need: a live list crosses 15 rows before anything else does.
`commitments.md` — `## Commitments`, same columns, both directions in one table.
`goals.md` — `## Goals`, same columns; a closed goal is kept for one quarter, then deleted.
`habits.md` — `## Habits`, same columns.
`someday.md` — `## Someday`, one line per parked idea with the date it was parked; anything older than a year is offered for deletion at the quarterly reset.

## Legacy layout

An older version of this skill created a folder tree. If it exists, map it rather than leaving it to rot — keep the originals until the user confirms the move, and write a `## Boxes` line for anything that becomes a box:

`dashboard.md`, `goals/active.md` → `## Goals` · `goals/someday.md`, `someday/ideas.md` → `## Someday` · `projects/active.md`, `projects/waiting.md` → one file per project in the shared projects box · `tasks/next-actions.md`, `tasks/this-week.md` → `## Tasks` · `tasks/waiting.md`, `commitments/promises.md`, `commitments/delegated.md` → `## Commitments` · `tasks/done.md` → the current `reviews/<year>.md`, summarised rather than copied line by line · `habits/active.md` → `## Habits` · `habits/friction.md`, `focus/distractions.md` → `## Friction` · `planning/*`, `routines/*` → `artifacts/` if they hold a real routine · `focus/sessions.md` → `sessions/<year>.md` · `reviews/weekly.md`, `reviews/monthly.md` → `reviews/<year>.md` · `inbox/capture.md`, `inbox/triage.md` → `## Tasks` or `## Someday`, one item at a time.

Empty scaffolding files are deleted, not migrated: they are the reason the old layout stopped being read.
