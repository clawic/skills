# Worktrees — Two Checkouts, One Repository

A worktree is a second (third, tenth) working directory backed by the same object store. No second clone, no re-fetch, no duplicated history — and no stashing to switch context.

## When A Worktree Beats A Stash

- Urgent hotfix while a refactor is half-finished: the refactor stays on disk, untouched and buildable.
- Reviewing or testing someone's branch without disturbing your index.
- Running a long test suite on `main` while you keep editing the feature.
- Building two revisions side by side to compare behavior or bisect a performance regression.
- Parallel agent or script tasks: each worktree has its OWN index and its own `index.lock`, so concurrent Git operations do not collide the way two processes in one checkout do.

## Core Commands

```bash
git worktree add ../hotfix -b hotfix main   # new dir, new branch, based on main
git worktree add ../review pr-branch        # check out an existing branch
git worktree add --detach ../probe v1.2.0   # detached at a tag; no branch consumed
git worktree list --porcelain               # scriptable inventory
git worktree remove ../hotfix               # refuses if dirty; --force overrides
git worktree prune                          # after someone rm -rf'd a worktree directory
```

## The Rules That Surprise People

- **A branch can be checked out in only one worktree.** `fatal: 'main' is already checked out at /path` is not a bug: use `--detach`, or create a branch. This is also why the main checkout cannot switch to a branch a worktree holds.
- **`.git` is a FILE in a linked worktree**, containing `gitdir: /main/repo/.git/worktrees/<name>`. Tools that test for a `.git` directory, copy `.git` around, or archive the folder break here.
- **Config is shared, not copied.** Every worktree reads the same `.git/config`, so a per-worktree `user.email` or sparse-checkout setting needs the worktree-config extension: `git config extensions.worktreeConfig true` (git >=2.20) once, then `git config --worktree <key> <value>`.
- **Hooks are shared too** (same `core.hooksPath`), but `$GIT_DIR` differs per worktree — a hook that hardcodes `.git/` misbehaves in linked worktrees. Use `git rev-parse --git-common-dir` for repo-wide paths and `--git-dir` for worktree-local ones (`hooks.md`).
- **Objects are shared; build outputs are not.** `node_modules`, `.venv`, `target/`, and IDE indexes are per-directory — the real cost of a worktree is a dependency install and a language-server reindex, not disk for history.
- **Submodules are not initialized automatically** in a new worktree: run `git submodule update --init` there too (`submodules.md`).
- **Deleting the directory is not removing the worktree.** The administrative entry stays under `.git/worktrees/` until `git worktree prune`, and until then the branch is still considered checked out.
- `git worktree lock ../on-usb` keeps prune from reaping a worktree on removable or network storage; `unlock` reverses it.

## Layouts That Scale

Bare repo as the hub, one worktree per active branch — the standard setup once you use worktrees daily:

```bash
git clone --bare git@host:org/repo.git repo/.bare
echo "gitdir: ./.bare" > repo/.git
cd repo
git worktree add main
git worktree add feature-x -b feature-x main
```

Every branch is a sibling directory, the object store exists once, and `git worktree list` is your branch dashboard. Fetch refspecs need setting on a bare clone (`git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'`), otherwise `git fetch` populates nothing useful.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `git worktree add ../x` with a relative path from inside another worktree | Resolved against the current directory, not the repo root — worktrees land in surprising places | Use absolute paths, or `add "$(git rev-parse --show-toplevel)/../x"` |
| Deleting a worktree directory and expecting the branch to be free | The stale entry keeps the branch locked | `git worktree remove`, or `prune` after the fact |
| Assuming `git stash` in one worktree is visible in another | The stash ref is repo-wide, but the working states belong to whichever worktree created them; applying elsewhere conflicts wildly | Commit to a temporary branch instead of stashing across worktrees |
| Running `git gc --prune=now` while a worktree is checked out elsewhere | Objects referenced only by another worktree's index have been reaped in older Git versions | Keep worktrees registered (never orphaned) so their refs and indexes count as reachable |
| One worktree per experiment, never cleaned up | Dozens of stale checkouts, each pinning a branch and a dependency tree | `git worktree list` weekly; remove on merge, like branches |
