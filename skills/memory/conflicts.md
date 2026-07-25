# Conflicts — Changed Facts, Wrong Facts, and Facts That Aged

The store's credibility is decided here, and the asymmetry is the whole reason: a miss is visible, so the user compensates for it; a confidently recalled stale fact is invisible, so the user acts on it.

## The Two Questions, In Order

```
Was it true on the day it was written?
  NO  → wrong fact      → delete the line (and the file, if the whole entry was wrong)
  YES → changed fact    → supersede: new dated line on top, old line to History with its original date
        Not sure?       → keep it, mark `stale YYYY-MM-DD`, and say so at recall
```

Deleting a changed fact destroys the timeline; archiving a wrong fact resurrects it as truth (SKILL.md Rule 8). The question above is the only thing separating them, and the source marker (Rule 4) is usually the evidence: an `inferred` line that turned out false was wrong; a `stated` line that turned out false has usually just changed.

## Superseding, Concretely

```markdown
## Facts
- 2026-07-25 · stated · Head of Product at Northwind

## History
- 2024-11-08 · stated · PM at Acme   (superseded 2026-07-25)
```

- The new line is added, the old one **moved**, not edited in place — an edited line loses its own date and the change becomes untraceable.
- Both dates stay: when the fact started being true, and when the store learned it changed. Those are different, and users ask about both.
- Update `Updated:` in the header and the date column in the category INDEX in the same pass, or recall keeps preferring a fresher-looking twin.
- Inbound links (`→ people/alice-smith.md`) survive supersession untouched. That is the point of Rule 5.

## Precedence When Two Facts Disagree

Apply in order; stop at the first that decides:

1. **The user's live statement** beats everything stored, always. They are the source; the store is a copy.
2. **Later date** beats earlier date.
3. **Same date** → `stated` beats `observed` beats `inferred`.
4. **Still tied** → the entry in the fact's canonical home (Rule 5) beats the copy elsewhere, and the copy gets converted to a link.

An `inferred` line never silently overwrites a `stated` line, whatever its date. If an inference contradicts something the user said, that is a question to ask, not a write to make.

## Expiry by Fact Class

Nothing here expires automatically. This table is what the monthly staleness sweep checks against, and it is a *re-check* pass, never a deletion pass:

| Class | Examples | Re-check when |
|---|---|---|
| Identity | Birthday, where someone grew up, a project's founding date | Never; only correct if wrong |
| Preference | Communication style, dietary constraint, tooling taste | Two years, or on contradicting evidence |
| Role and place | Job title, employer, city, phone | One year — the single largest source of embarrassing recalls |
| Commercial | Prices, rates, contract terms, headcount | Six months, or at the contract's own date |
| State | Project status, blockers, next steps, "currently reading" | Weeks; a `Next Steps` list untouched for a quarter is fiction |
| Relationship | Who reports to whom, who owns what | On any role change of either side |
| Anything unclassified | — | Treat as State: the shortest window, because an unclassified fact is usually a status |

A fact whose re-check window has passed is not wrong — it is *unverified*. Recall it with its date attached rather than deleting it.

## Correcting the Agent's Own Record

When the user says "no, that's not what I said":

1. Open the line and read it back verbatim with its date and source.
2. If it was `inferred`, delete it — an inference the user rejects has no residual value.
3. If it was `stated` and the user disputes ever saying it, replace it with the correction and note `corrected 2026-07-25`; do not argue the transcript.
4. Check whether the wrong line propagated: `grep -ril` the distinctive term across the store and fix every copy (Rule 5's failure mode).

## Duplicate Facts Across Files

The silent version of a conflict: the same fact in two files, updated in one.

```bash
grep -ril "northwind" ~/Clawic/data/memory/ | head
```

Resolution: pick the file the fact is most *about* as the canonical home, keep the newest version there, and replace the other copies with a link line. Never keep "the more detailed one" in both places — detail is exactly what diverges.

## Deletion Discipline

- With `delete_policy: confirm`, name what will be removed and wait; with `direct`, delete and report it after.
- A deletion is three operations, always together: the line or file, its index row, and any inbound `→ path.md` pointing at it. The third is the one that gets skipped, and a dead pointer is both a broken answer and a leftover trace of the deleted name.
- Deleting a whole entity is a privacy operation as much as a hygiene one: report back exactly what was removed, by path. "Done" is not a receipt.

## Back To

SKILL.md — Core Rules 4, 5 and 8 (source markers, one home, supersede vs delete), When Facts Change (the user-facing table this file expands), Output Gates (the pre-delete check).
