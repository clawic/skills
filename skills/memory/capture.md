# Capture — What Gets Written, and How

Capture failures are invisible: nobody notices the fact that was never written until the day it is needed. Cheap capture with a weekly sort beats a perfect taxonomy nobody feeds.

## The Durability Test

Write it if two of these three are true:

1. **Persistence** — it will still be true, or still matter, in 30 days.
2. **Cost of loss** — getting it wrong later costs the user time, money, or credibility (a client's constraint, a decision's reasoning, a person's preference).
3. **Non-derivable** — it can't be re-derived from files on disk or a quick search (a repo's URL is derivable; why the team rejected the alternative is not).

Explicit "remember this" bypasses the test — always write it. When exactly one condition holds and `inbox_enabled` is true, inbox it; the weekly sort decides.

## Store the Fact, Not the Conversation

| Heard | Bad entry | Good entry |
|---|---|---|
| "We're probably going with Postgres, Mongo scared us" | User mentioned maybe Postgres | 2026-07-25 · stated · Alpha uses Postgres; Mongo rejected over schema-migration risk |
| "Alice can't do mornings" | Alice is busy in the morning | 2026-07-25 · stated · Alice takes no calls before 10:00 (her timezone) |
| "Let's ship after the audit" | Ship later | 2026-07-25 · stated · Alpha launch gated on the security audit, not on a date |

Rules the table encodes: resolve hedges into a fact plus a confidence word; keep the *reason* (it is what makes the fact re-usable); keep the units and the qualifier (timezone, currency, scope) — a number without its unit is a future bug.

## Source Markers (Rule 4)

| Marker | Means | Recall treats it as |
|---|---|---|
| `stated` | The user said it in words | Authoritative **at equal date only** — it breaks ties, it does not outrank a later date (SKILL.md When Facts Change) |
| `observed` | Read from a file, output, message, or document | Trustworthy but re-checkable; a 2026 `observed` line beats a 2024 `stated` line |
| `inferred` | The agent concluded it from context | Provisional; never asserted back as if the user said it, and never silently overwrites a `stated` line whatever its date |

An `inferred` line that the user later confirms is rewritten as `stated` with the confirmation date. This is the whole reason the marker exists: without it, the agent's guesses become the user's history within a month.

## Atomicity

- One fact per line. A paragraph cannot be superseded or deleted surgically, and a half-wrong paragraph poisons everything around it.
- Compound statements split: "Alice moved to Northwind as Head of Product and now handles the EU accounts" is two lines — the role and the scope change independently.
- Long-form the user wrote (a brief, a spec, a voice-note transcript) goes in its own file with a pointer from the entry; entries stay scannable (Rule 7).

## Where It Goes, in One Pass

```
Does a domain skill already own this store?      → its ~/Clawic/data/<slug>/, pointer only here (Rule 5)
Is it about a named person or organization?      → people/ or orgs/
Is it about a named project or deliverable?      → projects/
Is it a choice with reasoning behind it?         → decisions/
Is it reusable knowledge independent of people?  → knowledge/
Is it an item in a set the user collects?        → collections/
Does it fit two of the above?                    → the one the user would search first; link from the other (Rule 5)
None of the above?                               → inbox/, one line, sort at maintenance
```

Never create a category to hold a single fact — the closest existing category plus a Keywords entry beats a folder with one file in it (Rule 2).

## Capture During Long Sessions

- Write at the moment the fact lands, not at the end. A summary written at session end has already lost the qualifiers.
- Batch only the INDEX update when several facts land in the same category in one minute — the entries themselves still go first (Rule 3).
- Never write a fact the user is still arguing with themselves about; wait for the sentence that settles it, then write that one.
- If the same subject comes up a third time in a session, it deserves an entry rather than three inbox lines.

## Capture From Documents and Pastes

- Store the extracted facts plus a pointer to the source, never a re-summary posing as the source: `observed · from contract-2026-07.pdf, clause 4`.
- Copy exact figures, dates, and names verbatim; paraphrase everything else. Paraphrased numbers are how a rounding error becomes a stored fact.
- The user's own writing (their bio, their voice, their constraints) belongs in its own file referenced from `config.yaml` by path, not inlined into an entry.

## What Not To Capture

| Skip | Why |
|---|---|
| Credentials of any kind | Rule 9 — plaintext store; write a pointer to the vault instead |
| Anything under `excluded_topics` | The user set the boundary; say so once, don't store, don't re-ask |
| Session mechanics ("user asked me to run tests") | Answers no future question |
| Facts about the agent's own behavior | That is the runtime's business, not the user's memory |
| A third party's sensitive details volunteered in passing | Health, finances, legal exposure, relationship details — store only if the user explicitly asks for it stored |
| Anything the store already holds | Update the existing line's date instead of adding a twin (Rule 5) |

## Index Row, Written In The Same Breath

An entry with no index row is invisible to every lookup above 50 files (SKILL.md Finding Things). The row carries only what a scan needs to choose the file:

```markdown
| Alpha | Active | Postgres/React | 2026-07-25 | alpha.md |
```

Status, the one distinguishing attribute, the date, the filename. Anything else belongs in the entry.

## Back To

SKILL.md — Core Rules (the write-before-reply gate, Rules 3 to 5), Entry Anatomy (the shape of what you write), Output Gates (the checks before replying).
