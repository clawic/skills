# Recovery — Getting Work Back

Order of operations when something is missing: **stop touching the repo**, do not run `git gc`, do not re-clone, then classify the loss. Classification decides everything.

## What Survives What

| State when it vanished | Recoverable? | Where it is |
|---|---|---|
| Committed (any branch, even deleted) | Yes | Reflog, then dangling commits |
| Staged, never committed | Usually | Dangling blobs via `git fsck` — content survives, filenames do not |
| Modified in the working tree, never staged | No, not in Git | Editor local history (JetBrains Local History, VS Code Timeline), backups, open buffers |
| Untracked, killed by `clean -f` | No, not in Git | Same as above, or filesystem-level recovery |
| Committed then `filter-repo`'d away | Yes, from a backup clone | `filter-repo` expires the reflog on purpose (see below) |

The asymmetry is the argument for staging early: `git add` alone converts an unrecoverable loss into a recoverable one.

## Reflog

- `git reflog` is HEAD's movement log: every checkout, commit, reset, rebase step, merge. `git reflog show <branch>` is that branch's own log.
- Read it as a timeline, not a history: `HEAD@{5}` is "five moves ago", unrelated to `HEAD~5` (five commits back). Confusing the two resets to the wrong place with total confidence (SKILL.md Revision Syntax).
- Time syntax works: `git reflog show main@{2.hours.ago}`, `git diff main@{yesterday} main`.
- Expiry: 90 days (reachable) / 30 days (unreachable). Both are `gc.reflogExpire` / `gc.reflogExpireUnreachable` and can be raised per repo.
- It is local and per-clone: it never pushes, dies with a re-clone, and a deleted branch's own reflog dies with the branch — search HEAD's reflog, which recorded the checkout anyway.

## Standard Rescues

```bash
git reset --hard ORIG_HEAD                 # immediately after a bad merge/rebase/reset
git reflog && git reset --hard HEAD@{7}    # the entry just before the damage
git branch rescue <sha>                    # name any commit you can see, before it can expire
git fsck --lost-found                      # dangling commits and blobs into .git/lost-found/
git fsck --unreachable | grep commit       # candidates for a dropped stash: git stash apply <sha>
```

- Dangling blobs have no name and no path: `git cat-file -p <blob> | head` identifies them by content, then redirect to the right filename.
- Dangling commits keep their tree: `git show <sha>` and `git checkout <sha> -- path` work normally.
- When several candidates look alike, sort by time: `git show -s --format='%ci %h %s' <sha>`.

## Situation Playbooks

**A file was deleted in some commit and nobody remembers which.**
```bash
git log --diff-filter=D --oneline -- path/to/file    # the commit that deleted it
git checkout <that-sha>^ -- path/to/file             # restore from its parent
```
If the path is unknown too: `git log --diff-filter=D --name-only --oneline | grep -i partname`.

**A rebase went wrong three steps in.** `git rebase --abort` while the rebase is still in progress. If you already ran `--continue` to the end: `git reset --hard ORIG_HEAD`, or the pre-rebase entry in the reflog.

**`git checkout .` / `git restore .` wiped edits.** Nothing in Git holds them. Check the editor's local history first — that is the only realistic path, and it works often enough to be the first question, not the last.

**The index is corrupted** (`error: bad index file`, `index file smaller than expected`): `rm -f .git/index && git reset` rebuilds it from HEAD. Cost: staged-but-uncommitted staging is lost; file contents in the working tree are untouched.

**Object database corruption** (`fatal: bad object`, `object file is empty`) after a crash or a full disk: `git fsck --full` names the broken objects. If a healthy clone exists anywhere (a teammate, a CI cache, a mirror), fetching from it repairs the missing objects; a single corrupt loose object can also be replaced by copying the same object file from that clone. Repairing in place beats re-cloning when you have local branches that were never pushed.

**Committed to a detached HEAD, then switched away.** The commit is unreferenced but alive: `git reflog` → `git branch rescue <sha>`. Unreachable objects are what the 30-day clock applies to.

**After `git filter-repo`.** It deliberately removes the origin remote and expires reflogs so you cannot keep both histories by accident. Recovery is only possible from the backup clone you made first — make one (`git clone --mirror . ../backup.git`) before any history-wide rewrite (`secrets.md`).

**A worktree was removed with `--force` while dirty.** Committed work is in the shared object store: `git reflog` finds it. Uncommitted work in that directory is gone (`worktrees.md`).

## What Actually Deletes Objects

- `git gc` (and the automatic one) prunes only unreachable objects older than the `gc.pruneExpire` grace period, `2.weeks.ago` by default. Automatic gc triggers on loose-object pressure, so a large rebase can start one behind your back (`large-repos.md`).
- The two clocks never compete: `gc` treats reflog entries as references, so while an entry survives (90/30 days, above) the commit it names is not unreachable and `gc.pruneExpire` cannot touch it. The two-week window governs only what no ref and no reflog entry names — dangling blobs from staged-but-uncommitted work, and whatever is left once a reflog entry has expired.
- `git gc --prune=now` and `git reflog expire --expire-unreachable=now --all` remove the safety net immediately. Never run either while recovering — that includes "cleaning up" before asking for help.
- Explicit branch deletion does not delete commits; it deletes a name. The commit dies later, quietly, at prune time.

## Prevention That Costs Nothing

- `git branch backup/$(date +%F-%H%M)` before any rewrite (SKILL.md rule 3).
- `git bundle create ../repo-$(date +%F).bundle --all` is a single-file snapshot of every ref, restorable with `git clone repo.bundle` — the offline backup for a repo that only exists on your laptop (`remotes.md`).
- Push the branch, even unfinished, to a personal remote namespace: the only backup that survives a dead disk.
- Raise the clock on repos where recovery matters: `git config gc.reflogExpireUnreachable 90.days` — record the retention you settle on under the thresholds preference area.
