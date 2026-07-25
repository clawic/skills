# Maintenance — Cadences, Sweeps, and Integrity Repair

Memory stores fail by accumulation, not by accident: unfiled inbox items, index rows that no longer match files, facts nobody re-checked. Every routine below is short, and skipping them is what makes the store feel unreliable three months in.

## Cadences

Frequency follows `maintenance_cadence` (default weekly).

**Weekly — 4 steps:**
1. Sort `inbox/` into categories; anything still homeless after two cycles gets filed or deleted.
2. Update every INDEX touched during the week (dates and new rows).
3. Archive items that reached terminal status (completed, cancelled, inactive).
4. Sweep for sync conflict copies if the store is synced, and union their rows rather than picking one.

**Monthly — 4 steps:**
1. Index size audit; split anything past `index_split_at`, along the axis you retrieve by.
2. Staleness sweep by fact class (below).
3. Integrity check: unindexed files, dead index rows, orphan links (below).
4. Delete facts that turned out wrong; supersede facts that changed (SKILL.md Rule 8).

**Quarterly:** re-read the root INDEX as a stranger would. Categories that never grew were the wrong categories (merge them); categories nobody queries are archive candidates.

## The Staleness Sweep

Not a deletion pass — a *re-check* pass. Facts don't rot, they just stop being verified, and each class carries its own re-check window (SKILL.md When Facts Change, class-based expiry row).

```bash
# Entries not updated in a year, hot categories only
find ~/Clawic/data/memory/people ~/Clawic/data/memory/projects -name "*.md" -mtime +365
```

For each hit, decide in one line: still true (touch the date, note `re-checked YYYY-MM-DD`), changed (supersede), wrong (delete), or unknown (mark `stale YYYY-MM-DD` and let recall carry the caveat). Do not batch-delete on age — an untouched identity fact is perfectly good.

Priority order when the sweep is long: role and place facts, then commercial facts, then state facts. Identity and preference facts almost never repay a sweep.

## Integrity Check

Three failure modes, three checks. Each is a comparison between what the filesystem holds and what the index claims.

| Failure | Symptom at recall | Check |
|---|---|---|
| File exists, no index row | Found only by full sweep; invisible above 50 files | Compare `ls` against the INDEX rows |
| Index row, no file | "Not found" after the index promised it | Extract `*.md` names from the index, test each path |
| Link points at a deleted or renamed file | Dead reference mid-answer | grep for `→ .*\.md` targets, test each |

Fix, never rebuild, when the damage is a handful of rows: a regenerated index loses the status and keyword columns that make it worth reading. Rebuild only when the index is unusable, and then re-enrich the columns by opening each entry.

## Archive Discipline

Archive on **terminal status**, not on age. A two-year-old active client stays; a project completed last month goes.

```bash
mv ~/Clawic/data/memory/projects/old-thing.md ~/Clawic/data/memory/archive/projects/
# then: remove the row from projects/INDEX.md, add it to archive/INDEX.md
```

```markdown
| Item | Type | Archived | Reason |
|------|------|----------|--------|
| OldProject | project | 2026-01 | Completed |
```

Archive holds old-but-**true** content only. Content that was never true gets deleted — an archived falsehood resurfaces later as truth (Rule 8). Content that merely *changed* is superseded inside its entry, not archived. Archive indices are exempt from `index_split_at`: they are off the hot path.

## Health Snapshot

Run it before deciding the store needs restructuring — most "the memory is a mess" reports are one unsorted inbox and two stale indices.

| Metric | Command | Healthy |
|---|---|---|
| Total entries | `find ~/Clawic/data/memory -name "*.md" \| wc -l` | Matches the sum of the index rows |
| Index sizes | `wc -l ~/Clawic/data/memory/*/INDEX.md` | Every one under `index_split_at` |
| Inbox depth | `ls ~/Clawic/data/memory/inbox \| wc -l` | Emptied at the last cadence |
| Loose files in root | `ls ~/Clawic/data/memory/*.md` | Only `INDEX.md` and `config.yaml` alongside it |
| Oldest untouched entry | `find … -mtime +365` | Reviewed at the last monthly sweep |

## After a Bad Pass

If a maintenance pass went wrong (mass rename, over-eager deletion, an index rebuilt over a good one):

- Git-backed store: `git diff` names every file touched, and revert is one command.
- Cloud-synced store: the provider's version history holds the previous copies, per file.
- Neither: the damage is permanent. That asymmetry is the argument for versioning a store the user cares about, and it is worth telling them once, before the pass rather than after.

## Back To

SKILL.md — Maintenance (the cadence summary this file expands), Rules 6 to 8 (index cap, entry size, supersede vs delete), When Facts Change (the expiry classes the sweep checks), Configuration (`maintenance_cadence`, `index_split_at`). Depth on any single step, including the runnable one-liners: SKILL.md Quick Reference.
