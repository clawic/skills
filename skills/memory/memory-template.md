# Memory Templates

Copy these shapes verbatim. Every entry, whatever the category, carries the same four elements from SKILL.md Entry Anatomy: display-name heading, `Keywords:` aliases, `Updated:` date, and dated + sourced fact lines (`stated` / `observed` / `inferred`, Rule 4).

Contents: System Configuration · Root Index · Projects · People · Organizations · Decisions · Knowledge · Collections · Inbox · Sync Folder · Index Size Limits.

## System Configuration

Create `~/Clawic/data/memory/config.yaml`:

```yaml
created: 2026-07-25
owner: [name]

# Variables from SKILL.md Configuration — only the ones the user has stated
index_split_at: 100
entry_max_lines: 200
delete_policy: confirm
recall_citations: true
sync_from_builtin: false
excluded_topics: []
inbox_enabled: true
maintenance_cadence: weekly
search_backend: grep

categories:
  - projects/
  - people/
  - decisions/
```

Only keys declared in SKILL.md Configuration belong here. A key with no described effect is not a setting; it is a future contradiction.

Older installs may have a `config.md`: merge its values into `config.yaml` and delete it. Preferences the user states go in here the moment they say them — never asked for up front.

## Root Index

Create `~/Clawic/data/memory/INDEX.md`:

```markdown
# Memory Index

## Categories

| Category | Items | Updated | Index |
|----------|-------|---------|-------|
| Projects | 12 | 2026-07-22 | projects/INDEX.md |
| People | 45 | 2026-07-20 | people/INDEX.md |
| Decisions | 23 | 2026-07-22 | decisions/INDEX.md |

## Quick Stats
Total items: ~80
Last maintenance: 2026-07-15
```

## Projects

**Index: `~/Clawic/data/memory/projects/INDEX.md`**
```markdown
# Projects Index

| Project | Status | Stack | Updated | File |
|---------|--------|-------|---------|------|
| Alpha | Active | Postgres/React | 2026-07 | alpha.md |
| Beta | Paused | Python | 2026-01 | beta.md |

Active: 5 | Paused: 3 | Archived: 20
```

**Entry: `~/Clawic/data/memory/projects/{slug}.md`**
```markdown
# Project: Alpha

**Keywords:** alpha, Northwind portal, the client dashboard
**Updated:** 2026-07-25

## Overview
Status: active | paused | complete
Started: 2025-09-01
Stack: Postgres, React

## Facts
- 2026-07-25 · stated · Launch gated on the security audit, not on a date
- 2026-05-02 · stated · Uses Postgres; Mongo rejected over schema-migration risk

## Decisions
- 2026-05-02 · Database choice → decisions/2026.md#database-alpha

## Team
- Alice (PM) → people/alice-smith.md

## Current State
[Where things stand — the fastest-rotting section; date it]

## Next Steps
- [ ] [Action]

## History
- 2025-09-01 · stated · Kicked off as an internal tool   (superseded 2026-02-11)
```

## People

**Index: `~/Clawic/data/memory/people/INDEX.md`**
```markdown
# People Index

### Work
| Name | Role | Company | Updated | File |
|------|------|---------|---------|------|
| Alice Smith | Head of Product | Northwind | 2026-07 | alice-smith.md |

### Clients
| Name | Company | Updated | File |
|------|---------|---------|------|
| Bob Lee | ClientCo | 2026-06 | bob-lee.md |

### Personal
| Name | Context | Updated | File |
|------|---------|---------|------|
| Carol Diaz | Friend | 2026-04 | carol-diaz.md |

Total: 45 contacts
```

**Entry: `~/Clawic/data/memory/people/{slug}.md`**
```markdown
# Alice Smith

**Keywords:** Ali, PM, product manager, Acme, Northwind, alpha project
**Updated:** 2026-07-25
**Not to be confused with:** → people/alice-smith-legal.md

## Basic Info
Role: Head of Product
Company: Northwind
Relationship: work | client | personal
Last contact: 2026-07-25

## Facts
- 2026-07-25 · stated · Moved to Northwind as Head of Product
- 2026-03-02 · stated · No calls before 10:00 her time; prefers async updates
- 2026-01-14 · inferred · Likely owns the EU accounts (not confirmed)

## How We Know Each Other
[Context]

## History
- 2024-11-08 · stated · PM at Acme   (superseded 2026-07-25)
```

Filenames are frozen identifiers, not labels — renames break every inbound link (SKILL.md Entry Anatomy).

## Organizations

**Entry: `~/Clawic/data/memory/orgs/{slug}.md`**
```markdown
# Northwind Ltd

**Keywords:** Northwind, NW, the portal client
**Updated:** 2026-07-25

## Relationship
Type: client | vendor | employer | competitor
Since: 2025-09

## Facts
- 2026-07-25 · observed · Renewal date 2027-01-31 (from contract-2026.pdf, clause 2)
- 2026-04-10 · stated · Procurement needs two weeks for any new tool

## People
- Alice Smith (Head of Product) → people/alice-smith.md

## History
```

Commercial facts carry their own re-check dates (SKILL.md When Facts Change, class-based expiry).

## Decisions

**Index: `~/Clawic/data/memory/decisions/INDEX.md`**
```markdown
# Decisions Index

| Year | Count | File |
|------|-------|------|
| 2026 | 23 | 2026.md |
| 2025 | 89 | 2025.md |
```

**Entry: `~/Clawic/data/memory/decisions/{year}.md`**
```markdown
# Decisions — 2026

## 2026-05-02 · Database for Alpha

**Decision:** Postgres
**Options considered:** Mongo, Postgres, SQLite
**Reasoning:** Schema-migration risk with Mongo outweighed the flexibility
**Outcome:** [What happened, if known]
**Revisit:** If write volume passes the point where a single primary struggles
**Related:** → projects/alpha.md
```

The reasoning field is the reason this category exists — an outcome without its reasoning cannot be reused, and it is the single most-regretted omission at session end.

## Knowledge

**Index: `~/Clawic/data/memory/knowledge/INDEX.md`**
```markdown
# Knowledge Index

| Topic | Depth | Updated | File |
|-------|-------|---------|------|
| Machine Learning | Deep | 2026-07 | ml/ |
| Finance | Reference | 2025-12 | finance.md |
```

**Entry: `~/Clawic/data/memory/knowledge/{topic}.md`**
```markdown
# [Topic]

**Keywords:** [aliases, abbreviations, the user's own jargon]
**Updated:** 2026-07-25

## Core Concepts
- **[Concept]:** [Explanation]

## Sources
- [Source, with date]

## Open Questions
- [Still to learn]
```

Private jargon and codenames get an entry in `knowledge/glossary.md` and an alias in the `Keywords:` line of every entity they touch.

## Collections

**Index: `~/Clawic/data/memory/collections/INDEX.md`**
```markdown
# Collections Index

| Collection | Items | Updated | File |
|------------|-------|---------|------|
| Books | 156 | 2026-07 | books.md |
| Recipes | 45 | 2026-01 | recipes.md |
```

**Entry:** one table, one row per item — collections are the one category where many facts share a file, because the row *is* the fact.

```markdown
# Books

## Read
| Title | Author | Rating | Date | Note |
|-------|--------|--------|------|------|
| [Book] | [Author] | 5/5 | 2026-01 | [Takeaway] |

## To Read
- [Book] by [Author] — [Why interested]
```

Past ~100 rows the collection splits like any other category (Rule 6): by type, by year, by status.

## Inbox

**`~/Clawic/data/memory/inbox/{YYYY-MM-DD}.md`** — one file per day, one line per capture:

```markdown
- 2026-07-25 · stated · Northwind's procurement takes two weeks   → orgs/?
```

Capture cost must be near zero; the arrow is a guess for the sorter, not a commitment.

## Sync Folder (Optional)

**`~/Clawic/data/memory/sync/INDEX.md`**
```markdown
# Synced from Built-In Memory

| What | Source | Last Sync | File |
|------|--------|-----------|------|
| Preferences | MEMORY.md | 2026-07-22 | preferences.md |

One-way sync. Built-in memory is never modified.
```

## Index Size Limits

| Index Type | Max Entries | When Exceeded |
|------------|-------------|---------------|
| Root INDEX.md | 20 categories | More usually means categories overlap — merge before splitting |
| Category INDEX.md | `index_split_at` (default 100) | Split into subcategories (Rule 6) |
| Subcategory INDEX.md | `index_split_at` | Split again — each level under the cap multiplies capacity ×`index_split_at` |
| Archive INDEX.md | No limit | Rarely read; off the hot path |
| Any other index | `index_split_at` | Treat as a category index until it proves it is off the hot path |

## Back To

SKILL.md — Entry Anatomy (the four elements every template above carries), Configuration (every key `config.yaml` may hold), Rules 4, 6 and 7 (dates and sources, index cap, entry size).
