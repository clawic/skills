# Branching Traps

## Creation & Switching

- A new branch inherits your CURRENT commit — `git switch -c feature main` pins the base explicitly; branching from a stale or wrong branch bakes the wrong history into every commit that follows.
- `git branch feature` creates without switching — your next commits land on the old branch. Prefer `git switch -c feature`.
- Switching with uncommitted changes is inconsistent by design: Git carries changes over when no files overlap, blocks when they do. That "sometimes it works" is why `rebase.autostash` and explicit `git stash` exist. Untracked files always travel with you.
- Branch and tag sharing a name: in revision arguments Git resolves the TAG first (with an "ambiguous refname" warning) — a checkout you think targets the branch may target the tag. Rename one.
- `feature/x` creates directory `refs/heads/feature/` — after that, a branch named plain `feature` cannot exist ("cannot lock ref"). Pick flat or hierarchical naming, not both.

## Merge

- Fast-forward vs merge commit is not cosmetic: `--no-ff` gives you a revert handle — `git revert -m 1 <merge-sha>` undoes the whole feature in one commit; a fast-forwarded feature must be reverted commit by commit.
- Merging a long-lived branch produces one mega merge commit that is hard to review and hard to revert partially — rebase it into reviewable shape first (rule 5 in SKILL.md caps when).
- Reverting a merge poisons the re-merge (details → `history.md`, Revert section).
- Squash-merge orphans the source branch: keep committing on it and the next merge conflicts with your own squashed changes. Delete the branch after squash-merge; start fresh from main.

## Rebase

- Rebase rewrites SHAs; everything pointing at the old SHAs dangles — PR review comments, CI statuses, teammates' local branches. That is the entire reason for rule 1 in SKILL.md.
- Stacked branches: rebasing the bottom strands everything above it — `rebase.updateRefs` (git >=2.38) moves the whole stack; on older Git, `git rebase --onto` each branch manually.
- A conflict on commit 3 of 7 is not the end: resolve, `--continue`, and expect more — each replayed commit conflicts independently.
- `git rebase --skip` silently drops the current commit from history — legitimate only when the commit is genuinely obsolete; otherwise you just lost work with no warning.

## Tracking

- First push needs `-u` (or `push.autoSetupRemote`, → `commands.md`); without it, later `git pull`/`git push` error with "no upstream".
- Deleting a remote branch leaves both the local branch and its tracking ref: `fetch.prune` cleans the ref, `git branch -d` the branch.
- Listing safe-to-delete branches: `git branch --merged main | grep -v main` — but squash-merged branches never appear (their commits are unreachable from main); verify those in the PR UI and use `-D`.

## Naming

- Case-insensitive filesystems (macOS/Windows) collapse `Feature/x` and `feature/x` into one ref file — Linux CI then sees two different branches. Lowercase branch names, always.
- No naming convention on a long-running project = unfindable branches; `git branch --sort=-committerdate` mitigates, a `type/topic` convention prevents.
