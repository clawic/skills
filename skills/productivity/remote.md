# Remote Work — Boundaries Nothing Physical Enforces

The remote constraint: the office supplied a start, an end, a commute, a social baseline and visible evidence that you were working. Remove all five and each has to be rebuilt deliberately. The failure is rarely focus; it is that work has no edges and visibility has no default.

**Before advising**, read `## Constraints`, `## Energy Patterns` and `## Due` in `~/Clawic/data/productivity/memory.md`, plus `config.yaml` for `calendar_owned`. A distributed team across five timezones needs different advice from a solo remote worker in the same timezone as the office.

## What Fails Here

- **Always online.** A green dot from 08:00 to 22:00 advertises availability, not productivity, and it sets the team's expectation of your response time (`messages.md`).
- **Timezone martyrdom.** Taking the 06:00 and the 23:00 call because you can. Within a quarter the working day spans 17 hours and the middle of it is unusable.
- **No physical separation.** Laptop on the sofa, phone by the bed, chat on the watch. Work does not end; it just becomes ambient, and the ambient version prevents recovery without producing output.
- **Presence theatre.** Anxiety about invisibility produces performative activity — more messages, more updates, faster replies — which consumes the hours that would have produced the evidence.
- **Isolation by default.** Days without unscheduled human contact accumulate into something that feels like a motivation problem and is not.
- **Meetings as the only social contact.** Turning work meetings into the social baseline makes the meetings longer and the loneliness worse.

## What Works

- **Hard start and stop, written and visible.** 09:00 to 18:00, laptop closed, notifications off. A boundary others can see is the only one they can respect.
- **A commute substitute.** Ten minutes of walking at each end. It is the cheapest available transition ritual, and its absence is what makes evenings feel like an extension of the afternoon.
- **One place for work.** A room where possible, a specific corner and configuration otherwise. Never the bed, and ideally never the sofa — the association costs more than the comfort returns.
- **Async by default, sync by exception.** Write the document, record the walkthrough, ask in a thread. Every meeting avoided returns a block to several people at once (`meetings.md`).
- **Communication windows, published.** "I answer chat at 11:00 and 16:00" makes the rest of the day usable and is invisible to nobody once stated.
- **Visible outcomes, not visible hours.** A weekly written summary of what shipped, sent unprompted, replaces the ambient evidence the office used to provide and costs fifteen minutes.
- **Deliberate social contact.** A scheduled non-work call, a coworking day, an office day if there is one. Scheduled, because it never happens spontaneously.

## Timezones

- **Find the overlap and protect it.** With a 3-4 hour overlap, that window is for sync work only — decisions, unblocking, 1:1s — and everything else moves to writing. Spending overlap on solo work wastes the only shared resource the team has.
- **Rotate the pain.** Standing meetings at a fixed hour permanently tax the same region. Rotating them is a small logistical cost and a large fairness gain.
- **Write times explicitly, with zones**: "Tuesday 14:00 CET / 08:00 ET". The mental conversion is where the missed meetings come from.
- **Set an expectation of latency, not availability**: "I answer within one working day in my timezone." Without it, the person 8 hours ahead assumes you are ignoring them.
- **A decision that needs three timezones needs a document**, not a meeting. The meeting will exclude someone, and the excluded person will reopen the decision later.

## Visibility Without Theatre

- **Weekly written update**: what shipped, what is next, what is blocked. Three bullets. It is the single highest-return remote habit for anyone worried about being invisible.
- **Work in the open.** Draft in a shared document, ask questions in a public channel rather than a DM. The work becomes its own evidence, at no extra cost.
- **Claim outcomes, not effort.** "Shipped X, it does Y" beats "spent three days on X". Remote managers see outputs and infer effort; they cannot see the effort directly.
- **Do not confuse response speed with reliability.** Reliability is doing what you said by when you said. Fast replies with slipping commitments is the worst combination available.

## Hybrid Specifically

- **Match the task to the location.** Office days for collaboration, 1:1s and the social baseline; home days for deep work. Doing solo work in the office and taking calls at home inverts the advantage of both.
- **Do not schedule deep blocks on office days.** They will be interrupted, and the interruption is the point of being there.
- **Anchor the week.** Fixed office days beat flexible ones for planning, and for actually overlapping with anyone.
- **Beware the proximity gap.** Colocated colleagues get informal information and informal credit. The countermeasure is the written update and deliberate contact, not more hours.

## The Real Issue

Remote productivity problems are boundary and visibility problems, not focus problems. Nothing physical stops work, so it has to be stopped deliberately; nobody sees the work, so it has to be shown deliberately. Both are structural, both are cheap to fix, and both get misdiagnosed as personal discipline — usually by the person suffering them.

## What to Write Down

- Working hours, overlap windows, office days and timezone go to `## Constraints`. Timezone and locale are read from `~/Clawic/profile.yaml` when it exists, and this skill does not write there.
- Communication windows and the weekly update become `## Due` rows, with the response contract recorded in `config.yaml` under the `conventions` preference area.
- The startup and shutdown rituals, once they stabilize, go to `~/Clawic/data/productivity/artifacts/shutdown-routine.md` with its `## Boxes` line.
- Distributed colleagues go to `~/Clawic/data/contacts/contacts.md` with their timezone in the context column — it is the field that prevents most scheduling mistakes.
- An isolation or boundary pattern that recurs goes to `## Friction`, with the countermeasure that worked.
