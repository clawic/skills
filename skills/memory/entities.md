# Entities — People, Organizations, Projects

Every store converges on the same three subject types, and every store breaks on the same three problems: the same entity under two names, two entities under one name, and a rename that orphans every link.

## Filenames Are Identifiers, Not Labels

- Lowercase, hyphenated, stable: `alice-smith.md`, `northwind-ltd.md`, `project-alpha.md`.
- Derived from the name at creation time and then **frozen**. Alice marrying, Northwind rebranding, Alpha becoming Atlas — all of that changes the `# Display Name` heading and the Keywords line, never the filename.
- The display name lives in the H1; the aliases live in `Keywords:`. Retrieval reads those two, so a frozen filename costs nothing and saves every inbound link.
- Renaming anyway (the file name became actively misleading) is three operations: `mv`, then the category INDEX row, then every inbound link. All three, or recall reports "not found" over data that exists. Read the match list before rewriting anything — a bulk `sed` over the store is the one command here that can damage it silently.

## Two People, One Name

Never `alice-smith-2.md`. The disambiguator must be something the user would actually say when searching:

| Situation | Filename |
|---|---|
| Two Alices, different companies | `alice-smith-acme.md`, `alice-smith-northwind.md` |
| Two Alices, same company | `alice-smith-design.md`, `alice-smith-legal.md` (role) |
| Client and personal contact | Split by subcategory (`people/clients/`, `people/personal/`) rather than by filename |

Both entries carry a `**Not to be confused with:** → people/alice-smith-acme.md` line. That line is what stops a future session merging them by mistake.

## One Person, Two Files (Merging)

Discovery: an INDEX with two similar rows, or a full-text sweep returning two files for one name.

```
1. Canonical = the OLDER file (its slug is the one already linked elsewhere).
2. Append the other's Facts and History into it; sort by date; de-duplicate identical lines.
3. Union the Keywords lines — the alias that caused the split is the most valuable one in the store.
4. Replace the duplicate file with a single pointer line: "Merged into people/alice-smith.md (2026-07-25)".
   Keep the pointer for one maintenance cycle, then delete it once no links remain.
5. Remove the duplicate's INDEX row; update the canonical row's date.
```

The pointer step is not optional: deleting the duplicate outright breaks every link written while both existed, and those links are invisible until someone follows one.

## One File, Two People (Splitting)

Rarer and nastier: facts about two entities accumulated in one file (a shared alias, a company that split, a project that forked).

1. Create the second entry with its own slug.
2. Move lines one by one, keeping each line's original date and source — never re-date a moved fact.
3. Facts that genuinely apply to both get one home plus a link (Rule 5); "both" is usually a sign of a third entity (the relationship, the joint project) that deserves its own entry.
4. Add mutual `Not to be confused with:` lines.

## Relationships

Store the edge in the entity that *owns* it, and link from the other:

| Edge | Home | Link from |
|---|---|---|
| Alice is PM on Alpha | `projects/alpha.md` (the project owns its roster) | `people/alice-smith.md` |
| Alice reports to Bob | `people/alice-smith.md` (the person owns their position) | `people/bob-lee.md` |
| Northwind is Alpha's client | `projects/alpha.md` | `orgs/northwind-ltd.md` |

Rule of thumb: the edge lives where it changes least often. A roster changes when the project changes; a reporting line changes when the person changes.

Do not build a graph. Two hops of links answer nearly every real question; a third hop means the fact should have been written down directly.

## Entry Shape by Entity Type

All three carry the four elements of SKILL.md Entry Anatomy. What each type must not omit on top of those:

- **Person** — how the user knows them, communication preferences, the last interaction date. Without the last date, "should I follow up?" is unanswerable.
- **Organization** — the relationship (client, vendor, employer, competitor), the people entries that link here, and the commercial facts each carrying its own re-check date.
- **Project** — status, the decisions that shaped it (linked to `decisions/`), current state, next steps. Status and next steps are the fastest-rotting facts in any store; date them hard.
- **Domain-owned entity** — a pet, a plant, a household member, a board's projects. Those live in their own skill's `~/Clawic/data/<slug>/`; here they get one pointer line and nothing else (SKILL.md Rule 5).

## Naming the Store's Own Vocabulary

When the user has private jargon — a codename, an internal acronym, a nickname for a recurring meeting — record it once in `knowledge/glossary.md` and put the expansion in the Keywords line of every entity it touches. The vocabulary asymmetry described in SKILL.md Finding Things is mostly private jargon that was never written down.

## Back To

SKILL.md — Rule 5 (one fact, one home), Entry Anatomy (filename as stable slug, `Keywords:` as the retrieval index), Finding Things (why aliases decide whether an entity is findable at all).
