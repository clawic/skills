# Layouts — Folder Structures and Split Axes

The layout decides two things nothing else can fix: whether a lookup is two small reads or a sweep, and whether a human opening the folder can read the store like a book. Pick one at setup from the first fact's category, not from a taxonomy drawn in advance (SKILL.md Rule 2). Restructuring later is `mv` plus index updates — real work, but not a migration, so don't over-deliberate the first choice.

Contents: Choosing · Category-Based (default) · Domain-Focused · Time-Based · Hybrid Zones · Flat Plus Keywords · Splitting a Growing Category · Quick Capture Plus Inbox · Cross-References · Keywords Lines.

## Choosing

| User profile | Layout | The trap it carries |
|---|---|---|
| Mixed work and personal, several domains | Category-based (default) | Building the whole taxonomy on day one; empty folders rot and teach the user the system is dead weight |
| One profession dominates everything | Domain-focused | Personal items forced into work folders become unfindable — keep one `personal/` as pressure valve |
| Mostly journaling or event logging | Time-based | Reference facts filed by date are unfindable; "what did I decide about the database?" dies in a quarter folder |
| Strong current-versus-historical divide | Hybrid zones | Two placement questions per item (which category, which zone) — worth it only once flat categories feel crowded |
| No natural taxonomy: research pile, idea dump, clippings | Flat plus Keywords | The single INDEX hits `index_split_at` fast, and splitting it means inventing the taxonomy you avoided |
| Unsure, or anything else | Category-based | — |

## Category-Based (default)

Organized by type of information. Each folder appears when its first item arrives, never before.

```
~/Clawic/data/memory/
├── projects/
├── people/
├── decisions/
├── knowledge/
└── collections/
```

## Domain-Focused

Everything organized around one profession — sales, research, consulting, clinical practice. Retrieval matches the way the user already talks about their work.

```
~/Clawic/data/memory/
├── clients/
├── deals/
├── products/
├── competitors/
└── market-research/
```

## Time-Based

Only for journaling and event logs, where "when" *is* the retrieval key.

```
~/Clawic/data/memory/
├── 2026/
│   ├── q1/
│   └── q2/
└── archive/
```

Events by date; facts by subject. Mixed needs are the hybrid layout, not a compromise inside this one.

## Hybrid Zones

For heavy users who query current and historical context differently enough that the zone answers the question before the category does.

```
~/Clawic/data/memory/
├── active/           # Current focus
│   ├── projects/
│   └── people/
├── reference/        # Always relevant
│   ├── knowledge/
│   └── preferences/
└── archive/          # Historical
    └── 2025/
```

## Flat Plus Keywords

For piles where every placement decision is arbitrary and therefore wrong half the time.

```
~/Clawic/data/memory/
├── INDEX.md          # every entry, one row, with its keywords column
└── notes/            # one folder, every entry
```

Retrieval holds up fine — the `Keywords:` lines carry it. Browsing does not: a human cannot read a flat 400-entry folder and understand what is in it. Use it for one category inside a category-based store, not for the whole store.

## Splitting a Growing Category

At `index_split_at` (SKILL.md Rule 6), split along the axis you *retrieve* by, never alphabetically. Test first: ask three questions the user would actually ask of that category and see which attribute narrows all three.

| Category | Natural split axis |
|---|---|
| projects | status: `active/`, `paused/`, `archived/` |
| people | relationship: `work/`, `clients/`, `personal/` |
| decisions | year: `2026/`, `2025/` |
| knowledge | topic: `ml/`, `finance/` |
| Anything else | Whichever attribute the user names first when asked to describe the category out loud |

```
~/Clawic/data/memory/projects/
├── INDEX.md          # just points to the subdirs
├── active/
│   └── INDEX.md      # 20 entries
└── archived/
    └── INDEX.md      # 120 entries — acceptable: rarely read, off the hot path
```

Archive indices may exceed the cap. It protects everyday lookups, and an archive index is not on that path.

## Quick Capture Plus Inbox

```
~/Clawic/data/memory/
├── inbox/            # unsorted, capture-first
├── projects/
└── ...
```

Capture cost must be near zero: if filing requires thought, inbox it — a fact in the wrong folder beats a fact never written. Sort at the maintenance cadence and delete from inbox after filing. When inbox sorting keeps getting skipped because items "have no home", the pile is naming the category that is missing.

## Cross-References

```markdown
# ~/Clawic/data/memory/projects/alpha.md

## Team
- Alice (PM) → see people/alice.md
- Bob (Dev) → see people/bob.md

## Key Decisions
- Database choice → see decisions/2026.md#database-alpha
```

Direction rule: a fact lives in the file it is most *about* — Alice's phone in `people/alice.md`, even though it was learned during project Alpha — and everything else links. Updates hit the canonical file and every link stays true. Never paste content across files (SKILL.md Rule 5).

Two hops of links answer nearly every real question. A third hop means the fact should have been written down directly; this is a store, not a graph.

## Keywords Lines

```markdown
# ~/Clawic/data/memory/people/alice.md

# Alice Smith

**Keywords:** PM, product manager, Acme, alpha project, Ali, weekly sync
```

SKILL.md Finding Things explains why the line exists. What belongs on it, in descending value: the nickname the user actually says (`Ali`), the codename (`Project X`), the old company name, the abbreviation, and the recurring event the entity is attached to. Load it at creation and grow it every time a search misses.

## Back To

SKILL.md — Rule 2 (the user defines the structure), Rule 5 (one fact, one home), Rule 6 (index cap and the 100^depth arithmetic), Finding Things (the ladder these layouts have to serve), Configuration (`index_split_at`, `inbox_enabled`).
