# Submodules, Subtrees, and Vendored Code

## The Model (everything else follows from this)

A submodule is a single entry in the parent's tree with mode `160000`: a **pinned commit SHA**, not a branch, not "the latest". The parent repo records where the submodule was; nothing updates that automatically. Almost every submodule complaint is a consequence of expecting a branch and getting a pointer.

## Choosing Between The Three

| Option | The consumer must | Choose when |
|---|---|---|
| Submodule | Know it exists (`--recurse-submodules`, init, update) | Both repos are yours, they release independently, and you need exact-commit reproducibility |
| Subtree | Know nothing — the code is just there | Consumers must clone-and-build with no extra steps, and updates are occasional |
| Package registry | Add a dependency line | The code is a real library with versions — the answer people skip because Git tooling feels closer |

Subtree merges history into yours (bigger repo, noisier log, `git log -- lib/shared` still works). Submodule keeps histories separate (small repo, extra ceremony forever).

## Submodule Operations

```bash
git submodule add -b main <url> lib/shared     # -b records a branch for --remote updates
git clone --recurse-submodules <url>           # or, after the fact:
git submodule update --init --recursive        # fills the empty directories
git submodule update --remote lib/shared       # fetch the branch tip and MOVE the pointer
git submodule status --recursive               # -/+/U prefixes: uninitialized / moved / conflicted
git config --global submodule.recurse true     # pull, checkout, and switch recurse from now on
```

**The distinction that costs the most time:** `git submodule update` checks out the SHA the parent recorded (restoring reproducibility). `git submodule update --remote` fetches the configured branch and changes that recorded SHA (a content change you must commit in the parent). One restores state, the other creates a change.

## Traps

- **Empty directories after clone.** Submodules do not come with a plain clone. Symptom: build fails on missing files that are visibly "in the repo" on the web UI.
- **Detached HEAD inside the submodule.** Default checkout is detached at the pinned SHA. Commits made there belong to no branch and are trivially lost on the next `submodule update` — `git switch -c fix` inside the submodule before editing, always.
- **Push order is not optional.** Push the submodule first, then the parent. Reversed, the parent references a commit nobody else can fetch, and every teammate's `submodule update` fails. Enforce it: `git push --recurse-submodules=on-demand` (pushes both) or `=check` (refuses if the submodule commit is unpublished).
- **Switching branches leaves stale submodule contents** unless `submodule.recurse` is on — you build the new branch's parent code against the old branch's submodule, and the failure looks like a code bug.
- **`git status` in the parent says "modified: lib/shared (new commits)"** — that means the submodule's HEAD differs from the recorded SHA, not that files changed. `git diff --submodule=log` shows which commits, and it is the only readable form of a submodule diff.
- **Merge conflicts on a submodule pointer** show as a conflicted gitlink with no text to merge. Resolve by choosing a SHA: enter the submodule, check out the commit you want (often a merge of both sides), then `git add lib/shared` in the parent.
- **Removing a submodule takes three steps**, and skipping the third breaks a later re-add:
  ```bash
  git submodule deinit -f lib/shared
  git rm -f lib/shared                 # removes the gitlink and .gitmodules entry
  rm -rf .git/modules/lib/shared       # the hidden clone that survives everything else
  ```
- **URL changed upstream.** Editing `.gitmodules` is not enough: `git submodule sync --recursive` copies it into `.git/config`, which is what fetch actually reads.
- **Private submodules in CI** fail on auth, not on Git: the runner has credentials for the parent only. Use SSH with a deploy key that covers both, or rewrite protocols with `url.<https-base>.insteadOf` (`config.md`).
- **Stash does not cover submodules** — a stashed parent leaves the submodule wherever it was (`commits.md`).
- **Worktrees do not initialize submodules** — each new worktree needs its own `submodule update --init` (`worktrees.md`).

## Subtree Operations

```bash
git subtree add    --prefix=lib/shared <url> main --squash
git subtree pull   --prefix=lib/shared <url> main --squash
git subtree push   --prefix=lib/shared <url> main
git subtree split  --prefix=lib/shared -b extracted   # pull a directory out as its own history
```

- Be consistent with `--squash`: mixing squashed and unsquashed pulls produces conflicts Git cannot help with, because the two histories no longer share ancestry.
- Contributions flow upstream only through `subtree push` or a patch — local edits inside the prefix are ordinary commits in YOUR repo and invisible to the source project until pushed.
- Nobody but you needs to know a subtree exists: that is its entire advantage, and the reason it is preferable when your consumers are humans in a hurry.

## Merging Whole Repositories

Folding a repo into a monorepo while keeping history readable:

```bash
git clone <url> incoming && cd incoming
git filter-repo --to-subdirectory-filter packages/incoming
cd ../monorepo
git remote add incoming ../incoming && git fetch incoming
git merge --allow-unrelated-histories incoming/main
```

The `--to-subdirectory-filter` step is what keeps every historical path correct; merging first and moving files afterwards breaks `git log --follow` for every file in the imported project. The reverse operation (extract a directory into its own repo) is `git filter-repo --path packages/x --path-rename packages/x/:` (`remotes.md`).
