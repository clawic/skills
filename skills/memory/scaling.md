# Scaling — What Breaks First, and What To Do About It

grep over markdown is rarely the bottleneck anyone expects it to be — a full-text sweep returns paths, and paths are cheap. What breaks, in this order, is index freshness, then vocabulary, then the human's ability to browse. Plan against that order, not against file count.

## Growth Stages

| Store size | What works | What breaks first | Move |
|---|---|---|---|
| <50 files | Everything; a full scan is cheap | Nothing | Don't build structure you don't need yet |
| 50–500 files | Indices first, then one file | Stale index rows; entries nobody indexed | Index audit at every `maintenance_cadence` pass |
| 500–5,000 files | Hierarchical: root → category → file | Vocabulary mismatch — the fact is there, the word isn't | Keywords discipline (SKILL.md Finding Things) |
| >5,000 files | Two levels of ≤100 indices (100² = 10,000 capacity) | Human browsability; nobody can hold the taxonomy in their head | Third level, or a curated `INDEX.md` of the twenty things actually asked about |

The first three rungs are the search-ladder thresholds from SKILL.md Finding Things, unchanged; the fourth splits the `>500` rung by what the human, not the search, can still handle. The table adds what to fix at each size, never a different threshold.

## Capacity Arithmetic

Rule 6's cap is `index_split_at` entries per INDEX, and capacity = cap^depth. At the default of 100:

| Depth | Example path | Capacity | Reads per lookup |
|---|---|---|---|
| 1 | `people/INDEX.md` | 100 | 2 (root index + entry) |
| 2 | `people/clients/INDEX.md` | 10,000 | 3 |
| 3 | `people/clients/emea/INDEX.md` | 1,000,000 | 4 |

Every level costs one small read and buys a factor of 100. Depth 3 covers stores larger than any personal memory ever gets — running out of capacity is never the real problem; running out of *taxonomy* is.

## Splitting Under Pressure

Split along the axis you *retrieve* by, never alphabetically (SKILL.md Rule 6). At scale the axis is the easy part; the timing is what goes wrong.

- **Split too early** and every item costs a placement decision for no lookup benefit.
- **Split too late** — an index several times past `index_split_at` — and every lookup scans screens of rows for one hit.
- **Split wrong** and items land in two plausible subfolders, which is a duplicate-fact generator (Rule 5).

Archive subcategories are exempt from the cap: they are read rarely and scanned whole when they are.

## When Files Stop Being Enough

Two conditions, both required, before adding any index layer beyond markdown:

1. The store is past 500 files, **and**
2. Recall still misses after the Keywords line was fixed for the terms that missed (SKILL.md Finding Things).

One without the other means the fix is discipline, not machinery. When both hold, in order of cost:

| Option | Buys | Costs |
|---|---|---|
| A hand-curated `INDEX.md` of the 20 hottest subjects | Most of the speed, zero new machinery | One line of upkeep per hot subject |
| A generated tag index over the `Keywords:` lines | Cross-category retrieval by keyword | Regeneration at every maintenance pass, or it lies |
| The runtime's own semantic search over the folder | Fuzzy questions plain grep can't answer | Availability varies by agent; set `search_backend: semantic` only where it exists |
| An external embedding or SQLite index | Ranked fuzzy retrieval at any size | A second source of truth, a rebuild pipeline, and the end of "plain files, readable anywhere" |

The last row is one-way in practice. Take it only when the user has felt the pain, not in anticipation.

## Costs Worth Knowing

- **Reading is the expensive part, not searching.** A grep across the store returns paths; opening a 400-line entry to answer one question is what actually burns the budget. That is what Rule 7's `entry_max_lines` cap protects.
- **Index size beats file count.** File count never enters the read path; index length does, on every single lookup. An index kept under `index_split_at` costs the same at 500 files as at 5,000 — which is why the cap, not the total, is the number to watch.
- **Many small files beat few large ones** for this workload: retrieval opens exactly one, and supersession touches exactly one line.
- **The archive is free.** Moving terminal items out of hot categories shrinks every future lookup and loses nothing.

## Pruning Instead of Scaling

Before adding depth, check whether the store is large or merely uncleaned:

- Entries with terminal status still in an active category → archive.
- Facts whose class expired years ago and were never re-checked → staleness sweep.
- Categories under five entries that duplicate another category's scope → merge (Rule 2's counterpart: a category that never grew was the wrong category).
- `inbox/` items older than two maintenance cycles → file them or delete them; a permanent inbox is a decision the user already made.

## Back To

SKILL.md — Finding Things (the first three rungs this file extends), Rule 6 (index cap and 100^depth capacity), Rule 7 (entry size), Where Experts Disagree (plain files versus a database), Configuration (`search_backend`, `index_split_at`, `entry_max_lines`).
