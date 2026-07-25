# Troubleshooting — Symptom to Cause

Work symptom-first. Each chain is ordered by how often the cause turns out to be the real one, and every step is a check, not a guess. Stop at the step that reproduces the symptom.

## "I don't see that in memory" (but the user is sure)

1. **Ladder actually run?** Root INDEX → category INDEX → file → full sweep → `inbox/` and `sync/`. Most misses stop at rung 2 and were never a store problem.
2. **Vocabulary**, the most common real cause (SKILL.md Finding Things): the fact was written in different words. Try the canonical name, the nickname, the old company name, then a word stem. Found it? Add the missed term to that file's `Keywords:` line.
3. **Unindexed file** — a body hit with no index hit. The write was interrupted before the index row. Add the row, then run the unindexed-files check on that whole category.
4. **Wrong category**: check the two categories the user might have considered, plus `inbox/`.
5. **The store is not where you think it is.** A symlinked store (cloud folder, external volume) whose target is unmounted reports an empty store instead of an error: `readlink ~/Clawic/data/memory` and confirm the target exists before believing a negative.
6. **It lives in built-in memory**, never copied here. Read it there; sync it if it needs structure (Rule 1).
7. **It was never written.** Say what was searched, then offer to capture it now — the miss is the best capture moment.

## Recall Returns an Outdated Fact

1. `grep -ril` the subject: two files holding the same fact is the usual cause, one updated and one not (Rule 5). Merge to one home plus links.
2. Undated line → it can't be judged stale at all. Date it and mark the source (Rule 4).
3. Dated, old, still recalled flatly: the answer needed its date attached. Role, place, price, and status facts are always recalled with their date (SKILL.md When Facts Change).
4. Nothing swept it because no sweep runs: set `maintenance_cadence` and run the staleness pass.

## Recall Returns Something the User Never Said

1. Check the source marker. An `inferred` line asserted as fact is the cause in most cases — delete it, and re-read Rule 4's marker discipline.
2. No marker at all → the entry predates the discipline. Mark what can be reconstructed, delete what can't be trusted.
3. A paraphrase drifted: the fact was stored as a summary of a conversation rather than the resolved fact. Rewrite it in the fact form.

## Two Entries for the Same Person or Project

1. Confirm they are one entity, not two people sharing a name.
2. Merge into the **older** file; the newer slug may be nicer but the older one is the one already linked.
3. Union the Keywords lines, leave a pointer in place of the duplicate for one cycle, then delete it.
4. If they are two entities: disambiguate the filenames with a stable token and add mutual `Not to be confused with:` lines.

## The Same Fact Lives in Two Skills' Stores

1. Ask which domain the fact is *about*, not which skill was open when the user said it. The pet fact is `dog`'s, the household fact is `family`'s, the plant fact is `garden`'s (SKILL.md Rule 5).
2. Delete the copy in the wrong store, do not "keep both in sync" — that is the fork, restated.
3. Leave one pointer line where the copy was, naming the owning store's path.
4. Recurring drift means capture is routing on conversation context. Re-read the domain row of SKILL.md Quick Reference before the next write.

## Lookups Feel Slow

1. `wc -l $M/*/INDEX.md` — an index past `index_split_at` is scanned on every lookup (Rule 6). Split along the retrieval axis.
2. Entry past `entry_max_lines` opened for one question → move the History bulk out (Rule 7).
3. Full sweeps happening routinely → the indices aren't answering. That is a Keywords problem before it is a size problem.
4. Genuinely large store (>500 files) with all of the above already done → this is growth, not breakage; take the >500-files row of SKILL.md Quick Reference.

## Structure Feels Messy

1. Loose `.md` files in the store root: only `INDEX.md` and `config.yaml` belong there; everything else moves into a category.
2. Categories with one or two entries that never grew: they were the wrong categories — merge into the closest neighbor (Rule 2).
3. Every capture triggers a placement debate: the taxonomy fights the way the user thinks. Re-pick a layout rather than adding folders.
4. `inbox/` is permanent and growing: the pile names the category that is missing.

## INDEX Out of Date

1. Files with no rows, rows with no files, links to deleted paths — three checks, each a comparison between the filesystem and the index's claims.
2. Repair row by row when the damage is small; regenerating loses the status and keyword columns.
3. Recurring drift means writes are landing without their index update. Tighten the sequence: entry, then row, in the same breath (Rule 3).

## Sync Problems

1. **Conflicted copies**: `INDEX (conflicted copy).md`, `INDEX 2.md` — silent forks holding real facts. Sweep, union the rows, delete the copies.
2. **Facts missing on one device**: partial sync, not a missing fact. Check the sync client before answering "not found".
3. **Built-in sync stale**: it is manual and one-way by design (Rule 1). Re-run it and record the date in `sync/INDEX.md`.
4. **Two agents writing**: entries never collide, indices always do. One writer per category at a time.

## Confusion Between This Store and Built-In Memory

- Built-in: the runtime's `MEMORY.md` and workspace `memory/` — the runtime owns them, this skill only reads (Rule 1).
- This store: `~/Clawic/data/memory/` in the home directory.
- Duplicated content across both: keep the summary in built-in, the detail here, and let the summary point here (SKILL.md Built-In Memory vs This Store).

## Nothing Above Fits

Reproduce the failure minimally: pick one fact the user is sure about and walk it end to end — is the file there, is the index row there, does the term appear in the file, does a fresh grep find it? The rung where the chain breaks names the fix.

## Back To

SKILL.md — Quick Reference (the situation router, and the way to every deeper file), Finding Things (the ladder most of these chains test), Traps (the same failures stated as prevention), Core Rules (the rule each chain is enforcing).
