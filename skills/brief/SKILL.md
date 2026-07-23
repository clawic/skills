---
name: brief
slug: brief
version: 1.0.2
description: >-
  Condenses user-provided material into decision-ready briefs — executive, status,
  meeting prep, handoff, decision. User picks sources; skill structures the output.
homepage: https://clawic.com/skills/brief
changelog: Sharper briefing structure and examples
metadata:
  clawdbot:
    emoji: 📋
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: Brief
---

## When To Use

- User asks for an executive brief, TL;DR, or bottom-line summary of material they provide
- A recurring status or project update is due and the reader needs the delta, not a restate
- A meeting is coming up and the reader needs decisions-needed plus a prep checklist
- A transition, offboarding, or vacation cover needs a handoff with gotchas and open questions
- The reader must choose between options and needs a recommendation with tradeoffs
- A long report, thread, or doc pile must be condensed to what drives a decision

Not for: compressing content when there is no decision or action to serve — use `summarizer` for shorter-content-only needs.

## Data Storage

```
~/brief/
├── preferences.md    # Learned format preferences
└── templates/        # Custom brief templates
```

Create on first use: `mkdir -p ~/brief/templates`

## Scope

This skill:
- ✅ Structures information user provides into briefs
- ✅ Learns format preferences from explicit feedback
- ✅ Stores preferences in ~/brief/preferences.md

**User-driven model:**
- User specifies WHAT information to include
- User grants access to any needed sources
- Skill handles STRUCTURE and FORMAT

This skill does NOT:
- ❌ Access files, email, or calendar without user request
- ❌ Pull data from sources user hasn't specified
- ❌ Store content (only format preferences)

## Quick Reference

| Situation | Play |
|-----------|------|
| Reader must choose between options | Decision brief: recommendation first, 2-3 options including do-nothing (`templates.md`) |
| Recurring status update | Project brief: lead with the delta since last brief, never a full restate |
| Meeting coming up | Meeting brief: decisions needed + prep checklist; deliver the day before, not the morning of |
| Transition, offboarding, vacation cover | Handoff brief: gotchas and open questions outrank achievements |
| Source is huge (long report, thread, doc pile) | Select by decision-relevance, never proportionally to source length |
| Formal channel (exec email, external doc) | Same structure, strip emoji markers, plain headers (`dimensions.md` → tone/channel) |
| User reacts to a delivered brief | Record it in `~/brief/preferences.md`; signal-to-dimension mapping in `dimensions.md` |
| Anything else | Executive structure: bottom line, 3 key points, explicit ask |

Full section-by-section templates for every type: `templates.md`.

## Core Rules

### 1. User Specifies Sources
When user requests a brief:
1. User provides the information OR specifies where to get it
2. If source requires access, user grants it explicitly
3. Skill structures and formats the output

Example:
```
User: "Brief me on project X status"
Agent: "I'll need access to the project docs. Can you share
        the status doc or grant access to the project folder?"
User: [shares doc or grants access]
→ Brief generated from user-provided source
```

### 2. A Brief Is Not a Summary
A summary compresses content; a brief serves the reader's next action. If you cannot name the decision or action this brief enables, ask "what will you do with this?" before writing. Competent people summarize; briefers select.

### 3. Write in Reverse Reading Order
Compose ask → bottom line → key points → context; the reader consumes the opposite order. Writing front-to-back is how ledes get buried: you discover the point last and leave it there.

### 4. Bottom Line Commits
One sentence, no hedge. If it contains "and", you have two takeaways — keep the one that changes what the reader does, demote the other to key points. "Mostly on track, but..." is not a bottom line; it is a decision you refused to make.

### 5. Three Key Points, Five Hard Max
A point qualifies only if removing it would change the reader's decision. Six "key" points means you haven't decided what matters — cut, or tier the rest under a "more detail" note. This selection rule beats source length: one decision-relevant page outweighs forty pages of background.

### 6. Every Number Carries a Comparator
"$1.2M revenue" is decoration; "$1.2M, 8% under plan" is a brief line. Valid comparators: target, prior period, benchmark. None available → write "no baseline yet" or drop the number.

### 7. Status Words Have Definitions
- **On track** — schedule/scope buffer intact
- **At risk** — buffer being consumed; the first slip counts, no grace period
- **Blocked** — cannot proceed without a named external action

Declaring at-risk early is cheap. A green-to-red flip with no warning destroys trust in every future brief you send.

### 8. Learn Only From Explicit Feedback
"Too detailed" / "missing X" / "perfect" → one line in `~/brief/preferences.md` with a level (`pattern`/`confirmed`/`locked`, promotion rules in `dimensions.md`). Check the file before writing any brief — a well-built brief in the wrong learned format still misses.

## Output Gates

Run before delivering any brief:
- [ ] Bottom line survives alone — if the reader stops there, they still have the takeaway
- [ ] Every metric has a comparator or an explicit "no baseline yet"
- [ ] The ask names an owner and a date, or states "no action needed" — never an implied ask inside an FYI
- [ ] Bad news sits above the fold, next to its mitigation, not in section four
- [ ] Data that can go stale carries source and as-of date
- [ ] Format matches `~/brief/preferences.md` — checked, not remembered

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Burying the lede | Readers triage; the point placed last is the point never read | Bottom line first — and write it first (Rule 3) |
| Hedged bottom line | Reader cannot act on a hedge | Commit to one takeaway; nuance goes in key points |
| Watermelon status (green outside, red inside) | The eventual flip costs more trust than early honesty ever would | First buffer slip → At risk (Rule 7) |
| Summarizing proportionally to source length | Background volume drowns the decision material | Select by decision-relevance (Rule 5) |
| Sandwiching bad news | In a brief, softening reads as hiding | Bad news first, mitigation beside it |
| Options without a recommendation | "Neutral" presentation transfers your analysis work to the reader | Recommend one; keep tradeoffs honest and visible |
| Exactly two options | Frames a false binary and hides the status quo | Price do-nothing as an explicit option (`templates.md` → Decision) |
| Emoji markers on formal channels | Reads unserious to exec/external audiences | Same structure, plain headers |

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/brief (install if the user confirms):
- `summarizer` - compression when there is no decision to serve, just shorter content
- `digest` - recurring curated updates pulled from external sources on a schedule
- `report` - recurring configured reports with fixed data sources
- `meetings` - the full meeting system (notes, agendas, follow-ups); brief covers only the pre-read

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/brief.
