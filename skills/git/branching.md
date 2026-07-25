# Branching Traps

## Creation & Switching

- A new branch inherits your CURRENT commit — `git switch -c feature main` pins the base explicitly; branching from a stale or wrong branch bakes the wrong history into every commit that follows.
- `git branch feature` creates without switching — your next commits land on the old branch. Prefer `git switch -c feature`.
- `git switch`/`git restore` (git >=2.23) split what `git checkout` overloaded: switching branches versus restoring files. The classic `git checkout <path>` that destroys working-tree changes with no warning is why the split exists.
- Switching with uncommitted changes is inconsistent by design: Git carries changes over when no files overlap, blocks when they do. That "sometimes it works" is why `rebase.autostash` and explicit `git stash` exist. Untracked files always travel with you.
- Branch and tag sharing a name: in revision arguments Git resolves the TAG first (with an "ambiguous refname" warning) — a checkout you think targets the branch may target the tag. Rename one.
- `feature/x` creates directory `refs/heads/feature/` — after that, a branch named plain `feature` cannot exist ("cannot lock ref"). Pick flat or hierarchical naming, not both.
- Detached HEAD is a state, not a failure: `git switch --detach v1.2.0` to test a release, and every rebase and bisect runs in one. Commits made there belong to no branch — `git switch -c <name>` before going anywhere else (`recovery.md`).
- Orphan branches (`git switch --orphan gh-pages`) start with no parent, but the index still holds the previous tree: `git rm -rf .` is the mandatory second step, or you commit the whole old project into the new root.

## Merge Or Rebase

| Situation | Choose | Why |
|---|---|---|
| Private branch, cleaning up before review | Rebase onto main | Reviewable commits, no merge noise, the SHAs are still yours to change |
| Branch is shared or already reviewed | Merge | Rebasing invalidates every reviewer's context and everyone's local copy (SKILL.md rule 1) |
| Same region conflicts across 3+ replayed commits | Merge | One resolution instead of N (SKILL.md rule 5) |
| Integrating under a linear-history policy | Rebase, then fast-forward | The policy is enforced server-side anyway; do it locally where conflicts are cheap |
| Long-lived branch whose integration date matters | `--no-ff` merge | The merge commit is both the audit record and the revert handle |
| Anything else | `integration_style`; on its `repo-default`, follow the repo | `git log --merges --oneline -20` reveals the house style faster than asking |

## Merge

- Fast-forward vs merge commit is not cosmetic: `--no-ff` gives you a revert handle — `git revert -m 1 <merge-sha>` undoes the whole feature in one commit; a fast-forwarded feature must be reverted commit by commit.
- Merging a long-lived branch produces one mega merge commit that is hard to review and hard to revert partially — rebase it into reviewable shape first (rule 5 in SKILL.md caps when).
- `git merge --no-commit --no-ff branch` stages the result for inspection first; `git merge --abort` then costs nothing.
- `git merge --ff-only` is the safe integration command for automation: it fast-forwards or fails, never inventing a merge nobody reviewed.
- Reverting a merge poisons the re-merge (details → `history.md`, Revert section).
- Squash-merge orphans the source branch: keep committing on it and the next merge conflicts with your own squashed changes. Delete the branch after squash-merge; start fresh from main.

## Rebase

- Rebase rewrites SHAs; everything pointing at the old SHAs dangles — PR review comments, CI statuses, teammates' local branches. That is the entire reason for rule 1 in SKILL.md.
- `git rebase --onto main old-base feature` moves a branch off an abandoned or already-merged base. Plain `git rebase main` replays everything since the merge-base — after a squash-merge that means replaying commits main already contains.
- Stacked branches: rebasing the bottom strands everything above it — `rebase.updateRefs` (git >=2.38) moves the whole stack; on older Git, `git rebase --onto` each branch manually.
- A conflict on commit 3 of 7 is not the end: resolve, `--continue`, and expect more — each replayed commit conflicts independently.
- `git rebase --skip` drops the current commit from history — legitimate only when the commit is genuinely obsolete; otherwise you just lost work, and nothing in the output says so.
- A rebase that appears to do nothing usually dropped patches Git recognized as already applied upstream. Correct behavior, alarming output: verify with `git log @{u}..` rather than re-running it.

## Tracking

- First push needs `-u` (or `push.autoSetupRemote`, → `commands.md`); without it, later `git pull`/`git push` error with "no upstream".
- Deleting a remote branch leaves both the local branch and its tracking ref: `fetch.prune` cleans the ref, `git branch -d` the branch.
- Listing safe-to-delete branches: `git branch --merged main | grep -v main` — but squash-merged branches never appear (their commits are unreachable from main); verify those in the PR UI and use `-D`.
- The reliable cleanup list once pruning is on: `git branch -vv | grep ': gone]'` — local branches whose upstream no longer exists (`remotes.md`).

## Naming

- Case-insensitive filesystems (macOS/Windows) collapse `Feature/x` and `feature/x` into one ref file — Linux CI then sees two different branches. Lowercase branch names, always.
- No naming convention on a long-running project = unfindable branches; `git branch --sort=-committerdate` mitigates, a `type/topic` convention prevents (the `branch_naming` variable in SKILL.md).
- Git rejects some names outright and shells eat others: no spaces, no `~^:?*[`, no leading `-`, no trailing `.lock`. `git check-ref-format --branch <name>` validates a name before a script creates it.
