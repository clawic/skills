# History Traps

## Reset

- `--soft` moves HEAD only (changes stay staged); `--mixed` also unstages; `--hard` also overwrites the working tree. Only `--hard` destroys anything.
- `reset --hard` casualties are asymmetric: committed work → reflog, trivially recoverable; staged-but-uncommitted → dangling blobs, `git fsck --lost-found`, tedious but possible; never-staged edits → gone forever. This ranking is why staging early is cheap insurance even without committing.
- `ORIG_HEAD` is set by merge, rebase, and reset: `git reset --hard ORIG_HEAD` is the one-command undo immediately after any of them.
- Untracked files ignore reset entirely — a "clean" `reset --hard` can still leave a dirty `git status`.

## Revert

- Revert creates a NEW commit; the original stays in history — that is the feature: it is the only safe undo on shared branches (SKILL.md, Recovery Playbook).
- Reverting a merge needs `-m 1` (keep the mainline parent). The hidden cost: re-merging that branch later is a no-op — Git considers its commits already merged. To land the feature again, revert the revert first.
- Reverting an old commit can conflict with everything committed since — past a point, a forward fix is cheaper than a revert.

## Amend & Rebase

- Both create new SHAs — "amend" is delete-and-recreate, not edit. Safe iff the commit shows in `git log @{u}..` (SKILL.md, rule 1).
- `--amend` with an empty stage edits only the message; with staged changes it folds them in. Check `git status` first so you know which of the two you are about to do.
- For fixing older commits, `git commit --fixup <sha>` + autosquash beats manual `rebase -i` gymnastics (→ `commands.md`).

## Reflog

- Local and per-clone: it never pushes and dies with a re-clone — reflog rescues only mistakes made in THIS clone.
- Expiry: 90 days (reachable) / 30 days (unreachable); a manual `git gc --prune=now` collects unreachable objects immediately — never run it while recovering.
- A deleted branch's own reflog dies with the branch — search `git reflog` (HEAD's log), which recorded every checkout and commit anyway.

## Rewriting Published History (last resort)

- Use `git filter-repo`, not `filter-branch` (deprecated, slow, footgun-laden) — for removing files or secrets across all history.
- After the rewrite: every collaborator must re-clone or hard-reset, open PRs break, and any leaked secret still needs rotation (SKILL.md, rule 7).

## Blame Archaeology

- Blame shows the last touch, not authorship: `-w` skips whitespace commits, `-C -C -C` follows code moved between files (→ `commands.md`).
- Mass-reformat commits poison blame permanently unless listed in `.git-blame-ignore-revs` (→ `commands.md`).
- When blame dead-ends, `git log -S "the exact code"` finds where the lines truly originated.
