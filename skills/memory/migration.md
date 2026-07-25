# Migration — Importing an Existing Pile, and Getting Out

Imports fail the same way every time: everything is copied in one pass, nothing has a date or a source, indices are written by hand once and never again, and the store is untrusted by week two. Import in slices, or don't import.

## The Slice Rule

One category per pass. Each pass ends with the same four things done, or the pass is not finished:

1. Entries written in this store's format (dated, sourced, one fact per line — SKILL.md Entry Anatomy).
2. Dedupe run against what already exists.
3. Category INDEX complete for that category.
4. A line appended to `~/Clawic/data/memory/MIGRATION.md`: what came from where, on what date, how many entries.

Start with the category the user queries most, not the largest one. The first slice has to prove the store answers questions; volume proves nothing.

## By Source

| Source | Where it lives | Import approach |
|---|---|---|
| Built-in agent memory | The runtime's MEMORY.md / workspace `memory/` | One-way into `sync/`, reformatted, date recorded (Rule 1) — never move it out of the runtime's control |
| Obsidian / plain markdown vault | A folder of `.md`, often with `[[wikilinks]]` | Nearly native: convert `[[Name]]` to `→ path/name.md`, add dates and Keywords, build the indices |
| Notion / Evernote export | Folder of markdown or HTML per page | Flatten one page to one entry; drop the export's ID suffixes and re-slug the filename lowercase-hyphenated |
| Apple Notes / Bear | Export to markdown or text first | Usually undated — use the export's modified date as the entry date, marked `observed`, never `stated` |
| Another agent's memory file | A single long file or JSON blob | Split by subject into entries; anything it recorded as an assumption imports as `inferred` |
| Chat logs and transcripts | Long conversations | Do NOT import wholesale. Extract facts against the durability test — a transcript is a source, not memory |
| A spreadsheet (contacts, inventory) | CSV | One row = one entry for people/projects; one row = one table line for collections |
| A domain a Clawic skill already owns (pets, garden, household, code style) | That skill's `~/Clawic/data/<slug>/` | Do not import it here at all — SKILL.md Rule 5; a pointer line is the whole migration |
| Anything else | — | Treat as chat logs: extract facts, drop the container |

## Dedupe, In Three Passes

Cheapest first; each pass catches what the previous one can't:

```bash
# 1. Filename collision — the obvious twins
ls ~/Clawic/data/memory/people/ | sort            # existing slugs, read against the incoming names

# 2. Title collision — same H1, different filename
grep -h "^# " ~/Clawic/data/memory/people/*.md | sort | uniq -d

# 3. Alias collision — the expensive, valuable one
grep -h "^\*\*Keywords:\*\*" ~/Clawic/data/memory/people/*.md
```

Write no scratch files into the store while doing this: a loose `.md` or `.txt` in a category folder is indistinguishable from an entry at the next maintenance pass. Keep the comparison in the terminal.

Collision found → merge into the existing entry, don't skip and don't overwrite. The imported copy is almost always the one with worse dates, so the existing entry stays canonical, and the alias that caused the split is the most valuable line in the merge.

## Dates and Sources for Imported Material

- Everything imported is `observed` unless the source records that the user said it.
- Use the source's own date when it has one. When it doesn't, use the file's modified date and mark it: `2026-03-11 (import date, original unknown)`. An invented date is worse than an admitted gap, because Rule 8 and the staleness sweep both trust dates.
- Import order does not matter, but recording the import date does: it is how a future session tells "old fact" from "recently learned old fact".

## What Not To Import

- Duplicates of things built-in memory already answers well (Rule 1).
- Dead projects and dead contacts — import them straight into `archive/`, or leave them out; an import is the cheapest moment to shed weight.
- Anything failing the durability test. Most of a long-lived vault is context that mattered on the day it was written — importing that wholesale reproduces the exact condition that made the old vault unusable.
- Credentials that the old system was storing (Rule 9). Import a pointer and tell the user their old notes hold live secrets.

## Exporting Out

The store is already the export: dated markdown, one subject per file, human-readable without this skill. What is worth doing before handing it anywhere:

- Freshen every INDEX first — indices are the only part that goes stale silently, and a stale index is what makes an export look incomplete.
- Flatten relative links if the destination won't preserve the folder tree.
- Strip `sync/` when the destination is another agent: those are copies of a runtime's private memory, not the user's authored facts.
- For a git-backed store, the history goes with it — including every fact deleted since day one. Say that out loud before handing the folder anywhere.

No lock-in exists by construction, and that is a feature to state when a user asks whether to commit to this system.

## Back To

SKILL.md — Entry Anatomy (the format every imported entry must land in), Rule 4 (dates and sources on imported material), Rule 5 (why a domain skill's data never comes here), Quick Reference (the import row).
