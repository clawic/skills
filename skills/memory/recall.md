# Recall — Finding It, and Knowing When It Isn't There

A store that holds the fact but can't surface it is worse than an empty one: the user stops trusting both answers, the hit and the miss.

## The Ladder, Rung by Rung

Thresholds are in SKILL.md Finding Things (<50 full scan · 50–500 indices first · >500 hierarchical). This is what each rung actually does.

1. **Root INDEX** — which categories exist. One read, always cheap, and it tells you which grep to run.
2. **Category indices** — `grep -i "term" ~/Clawic/data/memory/*/INDEX.md`. Matches the Keywords and status columns, not entry bodies.
3. **One file** — open the single best match. Two candidates? Open the one with the later `Updated:` date first.
4. **Full-text sweep** — `grep -ril "term" ~/Clawic/data/memory/` when the indices came back empty. This is a diagnosis as much as a search: a body hit with no index hit means the index is stale. Add the missing row before moving on, or the next lookup pays the sweep again.
5. **`inbox/` and `sync/`** — the two folders every ladder forgets. Unsorted captures and built-in copies live outside the category tree.

Stop at the first rung that answers. Escalating past rung 3 on a routine question means the Keywords line needs work, not that the store is too big.

## Query Expansion: 3 Variants Minimum

SKILL.md Finding Things sets the rule (≥3 variants before concluding absence, and the missed term goes into `Keywords:`). These are the variants worth spending them on, in order of hit rate:

| Variant | Example |
|---|---|
| The user's exact word | "the database thing" → `database` |
| The canonical name | `postgres`, `alpha` |
| The alias or nickname | `Ali`, `Project X`, the old company name |
| The adjacent entity | Can't find the decision? grep the person who made it, or the project it belongs to |

Fuzzy fallbacks that pay for themselves: drop to a word stem (`negotiat` matches negotiate/negotiation/negotiated), and search case-insensitively always (`-i`). A variant that produced the hit is worth more than the hit: it names the alias the entry was missing.

## Answering With a Recalled Fact

- Lead with the fact, then its date when the age matters: "Alice is at Northwind (as of 2026-07-25)". Age matters for role, address, price, and status facts; it does not for identity facts.
- With `recall_citations: true`, add the path (`people/alice-smith.md`). It is what lets the user correct the store instead of just correcting you.
- An `inferred` line is answered as an inference: "I noted, though you didn't say it directly, that…". Never launder it into a statement.
- Contradiction between two hits: answer with the later, stronger-sourced one (SKILL.md When Facts Change), mention the conflict exists, and resolve it in the same turn rather than leaving both.

## The Negative Result Protocol

Before "I don't have that":

1. All five rungs run, including `inbox/` and `sync/`.
2. ≥3 keyword variants tried.
3. Built-in memory checked — it may hold the summary while this store holds nothing.

Then say what was searched, not just that it failed: "Nothing under people/ or decisions/ for Northwind — want me to store what you just told me?" A miss is the best moment to capture, because the user is already thinking about the fact.

## Proactive Lookup (Without Being Asked)

Cheap, high-value triggers — index reads only:

| Trigger | Look up |
|---|---|
| A stored entity is named | Its entry, before answering anything about it |
| "As I mentioned" / "like last time" / "the usual" | The referenced subject; those phrases assert the store already knows |
| A decision resembling a logged one | `decisions/` — restating an old decision's reasoning is the store's highest-leverage moment |
| Work resuming on a named project | That project's entry and its Next Steps |
| The user contradicts a stored fact | The stored line, so the conflict is resolved instead of silently forked |
| Anything else | Nothing — no lookup |

Do not look up on: small talk, generic questions with no stored entity, or anything the user is clearly telling you fresh. Proactive reading costs context every turn, and memory reads must stay a minority of what a session loads.

## When Recall Keeps Missing

| Symptom | Cause | Fix |
|---|---|---|
| Body hit, no index hit | Index stale or never updated | Add the row, then run the unindexed-files check over that whole category |
| Nothing anywhere, user is sure | Written under a different name, or in `inbox/` | Search the adjacent entity; sweep inbox |
| Right file, wrong fact returned | Two lines about the same thing, different dates | Supersede properly: new line on top, old line to History with its own date (Rule 8) |
| Answer feels stale | Facts age by class and nothing swept them | Run the staleness sweep; recall role, place, price and status facts with their date attached |
| Every lookup needs the full sweep | Keywords lines are thin, or the taxonomy fights retrieval | Fix the `Keywords:` lines first; size is the cause only after that, and only past 500 files |
| Anything else | Unknown until reproduced | Walk one fact the user is sure about end to end: file there? index row there? term present in the file? The rung that breaks names the fix |

## Back To

SKILL.md — Finding Things (the thresholds and the ≥3-variant rule), Output Gates (what must run before "I don't remember"), Entry Anatomy (why `Keywords:` is the retrieval index).
