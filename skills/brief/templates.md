# Brief Templates

Standard structures per brief type. User preferences in `~/brief/preferences.md` override everything here. On formal channels (exec email, external docs), keep the structure and replace emoji markers with plain headers.

## Executive Brief

For leadership, stakeholders, or any reader with minutes, not hours.

```
⚡ BOTTOM LINE
[One sentence: the key takeaway]

📊 STATUS: [On track / At risk / Blocked]

KEY POINTS (3 max)
• [Most important thing]
• [Second most important]
• [Third if truly needed]

🎯 DECISION NEEDED
[What you need from them + by when — or "no action needed"]

📎 CONTEXT (optional)
[Only what changes how they read the points]
```

Sizing: one page ≈ 450-500 words ≈ a two-minute read at average silent-reading speed (~240 wpm, Brysbaert meta-analysis). Execs read on phones between meetings — if bottom line + ask don't fit the first screen, restructure before cutting words.

Key points: 3 max here — executive tightens the general five-max rule in SKILL.md Rule 5.

Context ordering, when included: SCQA (Minto) — Situation → Complication → Question; the Answer is your bottom line, already delivered above.

Worked example:
```
⚡ BOTTOM LINE
Q3 launch slips 2 weeks unless we cut the analytics module.

📊 STATUS: At risk

KEY POINTS
• Payments integration passed review — the critical path is now analytics
• Analytics is 3 weeks behind; vendor API changed mid-build
• Cutting analytics from v1 recovers the date; module ships standalone in Q4

🎯 DECISION NEEDED
Approve the cut by Friday, or accept the 2-week slip.
```

## Project Brief

For recurring status: sprint reviews, stakeholder updates.

```
📋 PROJECT: [Name] — [Date]

⚡ STATUS: [emoji] [On track / At risk / Blocked]

✅ COMPLETED (since last brief)
• [Done item]

🔄 IN PROGRESS
• [Current work] — [% or ETA]

🚧 BLOCKERS
• [Blocker] — [named unblocker] — [specific ask] — [days blocked]

📅 NEXT
• [Upcoming milestone] — [Date]

📊 METRICS (if applicable)
[Each number with its comparator: vs target, vs last period]
```

Principles:
- Recurring briefs report the delta. Repeating unchanged items trains readers to skim — and skimming readers miss the week something changes.
- A blocker line without a named person and a specific ask is a complaint, not an escalation. "Blocked 4 days" signals urgency better than any adjective.
- Status word definitions live in SKILL.md Rule 7; the first buffer slip is "At risk", not "On track with challenges".

## Meeting Brief

Pre-read sent before the meeting — the day before, not the morning of, or nobody reads it.

```
📋 MEETING: [Title] — [Date/Time]

🎯 PURPOSE
[Why this meeting exists — 1 sentence]

👥 ATTENDEES
[Key people and their role in this meeting]

📝 CONTEXT
[What changed since last time — delta only]

❓ DECISIONS NEEDED
• [Decision phrased as a question with options]

📎 PRE-READ (if any)
[Links, each with why it matters]

✅ PREP CHECKLIST
• [ ] [Item doable in minutes — longer items won't get done]
```

Principles:
- Phrase decisions as choices, not topics: "Choose vendor A or B" gets decided; "Vendor discussion" gets discussed.
- A meeting brief with no decisions-needed section is a signal the meeting could be this brief instead.

## Handoff Brief

For knowledge transfer, context passing, onboarding.

```
📋 HANDOFF: [Subject]

⚡ STATE
[Current state in 2-3 sentences]

🗺️ KEY CONTEXT
• [Important context]
• [Why things are the way they are — protects deliberate decisions from being "fixed"]

⚠️ GOTCHAS
• [Non-obvious thing that could bite them]

📌 PRIORITIES
1. [Most important next thing]
2. [Second]
3. [Third]

🔗 RESOURCES
• [Relevant doc]
• [Key contact for questions]

❓ OPEN QUESTIONS
• [Unresolved item + your current best guess]
```

Principles:
- Gotcha test: it cost YOU (or someone) real time. If it never bit anyone, it's context, not a gotcha.
- Open questions carry your best guess — a bare question transfers work; a question with a hypothesis transfers judgment.
- "Why things are the way they are" outranks achievements: the successor will be tempted to undo deliberate choices that look like mistakes.

## Decision Brief

For structured decision support.

```
📋 DECISION: [What needs to be decided]

⚡ RECOMMENDATION
[Your recommended choice — 1 sentence]

📊 OPTIONS (2-3, always including do-nothing)

**Option A: [Name]**
• Pros: [benefits]
• Cons: [drawbacks]
• Risk: [Low/Med/High — name the actual failure, not just the label]

**Option B: Do nothing**
• Cost of the status quo: [what it actually costs to wait]

⚖️ KEY TRADEOFFS
[What you're trading between options]

🎯 WHY [RECOMMENDATION]
[2-3 sentences]

⏰ DEADLINE
[Date + what is lost if it passes: "Decide by Fri or the vendor slot moves to Q4"]
```

Principles:
- Do-nothing is always an option and always has a price; presenting exactly two alternatives frames a false binary.
- Mark the decision reversible or irreversible (two-way vs one-way door, Bezos): reversible + cheap → short brief, recommend and move; irreversible → fuller evidence, and say why the extra length is there.
- A deadline without a consequence is a suggestion. State what expires.
