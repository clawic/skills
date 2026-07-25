# Sync — Multiple Devices, Multiple Agents, Version History

Plain files sync well; indices do not. Every problem in this file is the same problem: two writers, one INDEX.md.

## Cloud Folder Sync (Dropbox, iCloud, Drive, OneDrive)

Setup that keeps the declared path intact — the store stays at `~/Clawic/data/memory/`, the cloud folder holds the real bytes:

```bash
mv ~/Clawic/data/memory ~/Dropbox/clawic-memory
ln -s ~/Dropbox/clawic-memory ~/Clawic/data/memory
```

What to expect:

- **A broken symlink reads as an empty store, not as an error.** After a reinstall, a new machine, or a sync client that hasn't mounted yet, `~/Clawic/data/memory/` resolves to nothing and every lookup answers "no store" instead of failing loudly — and the next capture happily builds a fresh empty store on top of the dangling link. Before believing an empty store or creating one, check the link and its target: `readlink ~/Clawic/data/memory && ls ~/Clawic/data/memory/INDEX.md`. A dangling link is a mount problem, never a data-loss event; do not re-run setup until it resolves.
- **Conflicted copies are silent forks.** `INDEX (conflicted copy 2026-07-25).md`, `INDEX 2.md`, `INDEX-MacBook.md` — a fact written on the phone can sit in one of these for weeks. Sweep for them at every maintenance pass.
- **Entry files rarely conflict; indices always do.** Two devices adding two different entries both rewrite the same index. Resolution is a union: keep every row from both copies, sort, delete the conflicted file.
- **Partial sync breaks recall, not writes.** A device that hasn't finished syncing reports "not found" for facts that exist. Check the sync client before trusting a negative result.
- **Cloud history holds deletions.** A deletion request does not touch the provider's version history; say so when the request is made, not after.

## Git-Backed Memory

The strongest option for a single user who edits from more than one place, and the only one that makes a bad rewrite reversible.

```bash
git -C ~/Clawic/data/memory init
git -C ~/Clawic/data/memory add -A && git -C ~/Clawic/data/memory commit -m "memory: weekly maintenance 2026-07-25"
```

- Commit at maintenance cadence, not per write — per-write commits turn `git log` into noise and lose the one property worth having (a readable history of what changed).
- Merge conflicts land in indices; resolve as a union of rows, same as above.
- `git log -p people/alice-smith.md` answers "when did we learn this?" even for facts written before the date discipline was in place.
- Private repo only, and never a repo shared with code. The history is unencrypted and permanent — a secret committed once stays readable after deletion, which is the second reason Rule 9 declines them.

## Several Agents, One Store

Different agents (or several sessions of one) reading `~/Clawic/data/memory/` is the intended shape: plain markdown, no runtime assumptions, no lockfile.

| Operation | Safe concurrently? | Rule |
|---|---|---|
| Reading anything | Yes | Reads never conflict |
| Writing different entry files | Yes | Unique filenames, no shared state |
| Writing the same entry file | No | Last writer wins silently; the loser's fact is gone |
| Writing the same INDEX.md | No | The common collision — two rows, one survives |

Discipline that makes it safe in practice: **entries first, indices last**, and keep the index write in the same minute as the entry (a queued index update is an index update that never happens). When two agents genuinely run in parallel on the same category, let one own the writes and the other operate read-only for that session.

Never assume another agent's runtime memory is readable or writable — Rule 1 applies to every runtime, not just this one.

## One-Way Sync From Built-In Memory

```
~/Clawic/data/memory/sync/
├── INDEX.md          # what was synced, from where, when
├── preferences.md
└── key-decisions.md
```

Process: read built-in → reformat into this store's entry shape → write into `sync/` → record the date in `sync/INDEX.md`. Manual, one-way, and only for material that needs structure the runtime can't hold (Rule 1). Runs only when `sync_from_builtin` is true; the default is false.

Sync only what earns it. Copying everything creates two answers to the same question and no rule for which wins — the exact failure Rule 5 exists to prevent.

## Device-Specific Facts

Some facts are true on one machine only (paths, installed tools, local ports). They belong in the store, marked with their scope, not silently generalized:

```markdown
- 2026-07-25 · observed · Alpha's dev DB runs on the work laptop only, port 5433
```

An unscoped device fact is a fact that will be wrong on the other device — the same class of error as an undated one.

## Back To

SKILL.md — the store-location line in the intro (the declared path a symlink must preserve), Rule 1 (built-in memory is read-only, sync is one-way), Rule 5 (why copying everything creates two answers), Configuration (`sync_from_builtin`).
