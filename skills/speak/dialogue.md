# Dialogue — Questions, Confirmations, Recovery

Interactive voice turns: the user is present and waiting to talk. Different physics from a briefing — every second of agent speech is a second the user cannot answer.

## Turn Shape

- Verdict first (SKILL.md rule 2), one elaboration, then stop or ask. Two-sentence default turn.
- One question per turn, last and alone (SKILL.md Writing For The Ear). Two chained questions get one answer — the last one heard.
- Expect barge-in: front-load so an interrupted turn has already delivered its answer.
- Acknowledge before slow work: anything over about 2 seconds gets "Checking." first — silence on a voice channel reads as failure.

## Options and Choices

- At most 3 options aloud — a deliberate exception to the 4-item step cap (SKILL.md Quick Reference): choosing requires holding all options at once; following steps does not. More than 3 → speak your recommended one, offer the list in text.
- Name options with distinct first syllables: the user answers with a fragment, and "the standard plan" vs "the starter plan" is a mishearing factory.
- Always accept ordinal answers ("the first one", "the last one") — order memory outlives name memory.

## Confirmations

| Action class | Confirmation |
|---|---|
| Destructive, paid, or sent to others | Explicit echo of the exact payload: "Deleting all 14 drafts — confirm?" Never a bare "are you sure?" without the value |
| Reversible and internal | Implicit: state what you did and keep going ("Renamed to Q3 plan. Next...") |
| Data captured from speech (names, addresses, amounts) | Read back the captured value once, digits per SKILL.md rules, before acting on it |

Speech recognition upstream is lossy: an amount or address you heard is a hypothesis until echoed and accepted.

## Error Recovery (progressive, never verbatim repeats)

1. First miss: shorter rephrase of the question.
2. Second miss: rephrase plus an example answer ("Which month? March, for example.").
3. Third miss: fall back to text or on-screen options with a one-line notice (SKILL.md rule 8) — three spoken failures indict the channel, not the user.

When the answer does not fit the question (mishearing suspicion): confirm your interpretation before acting on it, and switch critical letters to NATO alphabet after one failed confirm (`normalization.md`).

## Anti-Habits

- No trailing "Anything else?" on every turn — only when closure is genuinely ambiguous.
- No re-greeting mid-session; the session has one hello.
- No narrating tool activity ("Now I'm calling the calendar API") — "Checking your calendar" is the ceiling.
- No apologizing more than once per failure; repeated apology consumes budget and trust equally.
