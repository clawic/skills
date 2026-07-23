# Organization Patterns

Choosing a layout:

| User profile | Pattern |
|--------------|---------|
| Mixed work + personal, several domains | 1. Category-based |
| One profession dominates everything | 2. Domain-focused |
| Mostly journaling / event logging | 3. Time-based |
| Strong current-vs-historical divide | 4. Hybrid |
| Unsure | Default: category-based. Restructuring later is `mv` + index updates, not a migration — don't over-deliberate. |

---

## Pattern 1: Category-Based (default)

Organize by type of information:

```
~/memory/
├── projects/
├── people/
├── decisions/
├── knowledge/
└── collections/
```

**Best for:** general use, multiple domains.
**Trap:** creating the full taxonomy on day one. Empty categories rot and teach the user the system is dead weight. Create each folder when its first item arrives (SKILL.md Rule 2).

---

## Pattern 2: Domain-Focused

Everything organized around one profession:

```
~/memory/
├── clients/
├── deals/
├── products/
├── competitors/
└── market-research/
```

**Best for:** one dominant domain (sales, research, consulting).
**Trap:** personal items forced into work folders become unfindable. Keep one `personal/` category as pressure valve.

---

## Pattern 3: Time-Based

```
~/memory/
├── 2026/
│   ├── q1/
│   └── q2/
└── archive/
```

**Best for:** journaling and event logs only.
**Trap:** reference facts filed by date are unfindable — recall works by *subject*, not by *when*. "What did I decide about the database?" dies in a quarter folder. Events by date; facts by subject. Mixed needs → Pattern 4.

---

## Pattern 4: Hybrid

```
~/memory/
├── active/           # Current focus
│   ├── projects/
│   └── people/
├── reference/        # Always relevant
│   ├── knowledge/
│   └── preferences/
└── archive/          # Historical
    └── 2025/
```

**Best for:** heavy users who query current and historical context differently.
**Cost:** every item now has two placement questions (which category? which zone?). Only worth it once flat categories feel crowded — not as a starting layout.

---

## Pattern 5: Growing a Category

Split at 100 index entries (SKILL.md Rule 6). Split along the axis you *retrieve* by, not alphabetically:

| Category | Natural split axis |
|----------|--------------------|
| projects | status: `active/`, `paused/`, `archived/` |
| people | relationship: `work/`, `clients/`, `personal/` |
| decisions | year: `2026/`, `2025/` |
| knowledge | topic: `ml/`, `finance/` |

```
~/memory/projects/
├── INDEX.md          # Just points to subdirs
├── active/
│   └── INDEX.md      # 20 entries
└── archived/
    └── INDEX.md      # 120 entries — acceptable: rarely read, off the hot path
```

An archive index may exceed 100 entries: it's scanned rarely, so the limit that protects everyday lookups doesn't apply.

---

## Pattern 6: Syncing from Built-In Memory

```
~/memory/sync/
├── INDEX.md          # what was synced, from where, when
├── preferences.md
└── key-decisions.md
```

**Process:** read built-in (MEMORY.md, etc.) → reformat for this system → write to `sync/` → record sync date in `sync/INDEX.md`.

One-way and manual (SKILL.md Rule 8). Sync only what needs deep structure here — built-in already answers its own quick queries; mass copying just creates divergence.

---

## Pattern 7: Quick Capture → Organize Later

```
~/memory/
├── inbox/            # Unsorted, capture-first
├── projects/
└── ...
```

Capture cost must be near zero: if filing requires thought, inbox it — a fact in the wrong folder beats a fact never written. Sort weekly; delete from inbox after filing.

**Signal:** if inbox sorting keeps getting skipped because items "have no home", the category set is wrong — the pile is telling you which category to create.

---

## Pattern 8: Cross-References

```markdown
# ~/memory/projects/alpha.md

## Team
- Alice (PM) → see people/alice.md
- Bob (Dev) → see people/bob.md

## Key Decisions
- Database choice → see decisions/2026.md#database-alpha
```

**Direction rule:** a fact lives in the file it's most *about* (Alice's phone → `people/alice.md`, even if learned during project Alpha); everything else links. Updates hit the canonical file and every link stays true. Never paste content across files (SKILL.md Rule 5).

---

## Pattern 9: Archiving

Archive on terminal status — completed, cancelled, went inactive:

```bash
mv ~/memory/projects/old-thing.md ~/memory/archive/projects/
# then: remove from projects/INDEX.md, add to archive/INDEX.md
```

```markdown
# Archive

| Item | Type | Archived | Reason |
|------|------|----------|--------|
| OldProject | project | 2026-01 | Completed |
```

Archive vs. delete: archive old-but-**true** content; delete content that's no longer true — an archived wrong fact resurfaces as truth later (SKILL.md, Maintenance).

---

## Pattern 10: Search Optimization

```markdown
# ~/memory/people/alice.md

# Alice Smith

**Keywords:** PM, product manager, Acme, alpha project, Ali, weekly sync
```

Why: grep matches the words used at *write* time, but recall happens in *today's* words. The Keywords line bridges that gap — load it with aliases the user actually says: nicknames ("Ali"), codenames ("Project X"), abbreviations, old company names. When a search misses and the file is later found, add the missed term to its Keywords line so the same search never misses twice.
