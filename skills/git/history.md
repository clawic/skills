# History — Undo, Rewrite, Transplant

## Undo Recipes

| Intent | Command | Leaves behind |
|---|---|---|
| Undo the last commit, keep the changes staged | `git reset --soft HEAD~1` | Nothing; the work is back in the index |
| Undo the last commit, keep the files, unstage them | `git reset HEAD~1` | Working tree untouched |
| Unstage one path | `git restore --staged <path>` | Index only; no destruction |
| Discard working-tree edits to one path | `git restore <path>` | Irrecoverable if never staged (SKILL.md rule 4) |
| Fix the last commit message | `git commit --amend` | A new SHA — unpushed only |
| Drop a file from the last commit | `git restore --source HEAD~1 --staged --worktree <path> && git commit --amend --no-edit` | A new SHA |
| Move the last 3 commits to a new branch | `git branch feature && git reset --hard HEAD~3` | The commits live on `feature` |
| Delete a commit from the middle (unpushed) | `git rebase -i <sha>~1`, mark `drop` | New SHAs from that point on |
| Undo anything already pushed | `git revert <sha>` | A new commit; history intact |
| Anything else | `git reflog` first, then the smallest tool that reaches the goal (SKILL.md rule 8) | — |

## Reset

- `--soft` moves HEAD only (changes stay staged); `--mixed` also unstages; `--hard` also overwrites the working tree. Only `--hard` destroys anything.
- `reset --hard` casualties are asymmetric: committed work → reflog, trivially recoverable; staged-but-uncommitted → dangling blobs, `git fsck --lost-found`, tedious but possible; never-staged edits → gone forever. This ranking is why staging early is cheap insurance even without committing.
- `ORIG_HEAD` is set by merge, rebase, and reset: `git reset --hard ORIG_HEAD` is the one-command undo immediately after any of them.
- Untracked files ignore reset entirely — a "clean" `reset --hard` can still leave a dirty `git status`.
- `git reset <sha> -- <path>` is a different operation despite the shared name: it stages that path as of that commit and never moves HEAD.

## Revert

- Revert creates a NEW commit; the original stays in history — that is the feature: it is the only safe undo on shared branches (SKILL.md, Recovery Playbook).
- Reverting a merge needs `-m 1` (keep the mainline parent). The hidden cost: re-merging that branch later is a no-op — Git considers its commits already merged. To land the feature again, revert the revert first.
- Reverting a RANGE: `git revert --no-commit A..B` then one commit, or `git revert -n` per commit — reverting them one by one in forward order fails as often as it works, because each revert changes the context of the next.
- Reverting an old commit can conflict with everything committed since — past a point, a forward fix is cheaper than a revert.

## Cherry-Pick

- `git cherry-pick <sha>` replays one commit's diff here, creating a new SHA. The same change now exists twice in the repo's history: fine across release lines, a duplicate-commit mess if the branch is later merged.
- `-x` appends "(cherry picked from commit ...)" — mandatory for backports, since it is the only durable evidence the two commits are the same change (`releases.md`).
- `git cherry-pick A..B` takes a range (exclusive of A); `A^..B` includes A. Off-by-one here skips the first commit with no warning.
- Conflicts stop the pick: resolve, `git cherry-pick --continue`, or `--abort`. `--skip` moves past a commit whose changes are already present.
- `-n` (no-commit) accumulates several picks in the index to be committed as one — the tidy way to backport three related commits as a single hotfix.

## Amend & Rebase

- Both create new SHAs — "amend" is delete-and-recreate, not edit. Safe iff the commit shows in `git log @{u}..` (SKILL.md, rule 1).
- `--amend` with an empty stage edits only the message; with staged changes it folds them in. Check `git status` first so you know which of the two you are about to do.
- For fixing older commits, `git commit --fixup <sha>` + autosquash beats manual `rebase -i` gymnastics (`commits.md`).
- Mid-rebase, `git status` tells you the state machine's position: which commit is being replayed and which of `--continue`/`--skip`/`--abort` applies. Read it instead of guessing; a rebase left half-finished blocks every other operation.
- `git rebase --exec 'npm test' main` runs a command after each replayed commit — the way to guarantee every commit in a branch builds, not just the tip.
- `git rebase --update-refs` (git >=2.38) carries the intermediate branch pointers of a stack along with the rebase.

## Reflog

- Local and per-clone: it never pushes and dies with a re-clone — reflog rescues only mistakes made in THIS clone.
- Expiry: 90 days (reachable) / 30 days (unreachable); a manual `git gc --prune=now` collects unreachable objects immediately — never run it while recovering.
- A deleted branch's own reflog dies with the branch — search `git reflog` (HEAD's log), which recorded every checkout and commit anyway.
- Full recovery procedures and what expiry actually deletes: `recovery.md`.

## Rewriting Published History (last resort)

- Use `git filter-repo`, not `filter-branch` (deprecated by Git's own docs, slow, footgun-laden) — for removing files or secrets across all history.
- Common invocations: `--invert-paths --path <file>` (delete everywhere), `--path <dir> --path-rename <dir>/:` (extract a subtree), `--to-subdirectory-filter <dir>` (nest a whole project), `--mailmap <file>` (fix historical identities), `--strip-blobs-bigger-than 10M`.
- Take a mirror backup first; filter-repo deliberately drops the origin remote and expires reflogs, so there is no undo inside the rewritten clone (`recovery.md`).
- After the rewrite: every collaborator must re-clone or hard-reset, open PRs break, and any leaked secret still needs rotation (SKILL.md, rule 7; full incident order in `secrets.md`).
- The cheapest alternative to a rewrite is often no rewrite: `.mailmap` fixes displayed identities, and a `git revert` removes content going forward without touching a single existing SHA.
