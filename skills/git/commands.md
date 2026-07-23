# High-Leverage Commands

Basics (`add`, `commit`, `push`, `pull`, `status`, `log`) are assumed. These are the flags and configs that separate fluent Git from muscle memory.

## One-Time Config That Prevents Whole Trap Classes

```bash
git config --global pull.rebase true            # pull never creates surprise merge commits
git config --global rebase.autostash true       # pull/rebase work on a dirty tree
git config --global rebase.autosquash true      # rebase -i honors fixup! commits automatically
git config --global rebase.updateRefs true      # git >=2.38: rebasing a stack moves dependent branches too
git config --global push.autoSetupRemote true   # git >=2.37: first push sets upstream, no more -u
git config --global fetch.prune true            # deleted remote branches disappear locally
git config --global rerere.enabled true         # identical conflicts auto-resolve the second time
git config --global merge.conflictStyle zdiff3  # git >=2.35: conflict markers include the base version
git config --global diff.algorithm histogram    # cleaner diffs on moved/refactored code
git config --global branch.sort -committerdate  # `git branch` lists recent work first
```

## History Forensics

```bash
git log -S "someFunction"            # pickaxe: commits that add/remove the string — when did this code appear/vanish
git log -G "regex"                   # commits whose diff matches regex (catches moves that -S ignores)
git log -L :myFunc:src/file.c        # full evolution of one function
git log --follow -p -- path          # file history across renames
git log --first-parent main          # PR-level history: one line per merge, branch noise hidden
git blame -w -C -C -C file           # ignore whitespace, follow code copied between files
git config blame.ignoreRevsFile .git-blame-ignore-revs   # git >=2.23: skip mass-format commits in blame
git range-diff main old-tip new-tip  # what actually changed between two versions of a rebased branch
```

## Surgical State Manipulation

```bash
git add -p                            # stage hunks, not files — split mixed work into clean commits
git restore -p                        # discard hunks selectively
git restore --source HEAD~3 -- path   # one file as it was 3 commits ago; nothing else moves
git commit --fixup <sha>              # mark a correction for a specific commit; autosquash folds it in
git stash push -m "msg" -- path/      # stash only some paths
git stash -u                          # include untracked — plain stash silently skips them
git cherry-pick -x <sha>              # record the origin sha in the message — traceable across branches
git rebase --onto main old-base feature   # move a branch off an abandoned or merged base
```

## Cloning Big Repos

```bash
git clone --filter=blob:none <url>    # blobless: full history metadata, file contents fetched on demand
git clone --depth 1 <url>             # shallow: fastest, but log/bisect/blame are truncated
```

The invisible distinction: blobless keeps every history command working (first access to old files is slower); shallow breaks them. CI throwaway checkout → shallow; a repo you will actually work in → blobless.

## Push/Pull Precision

```bash
git push origin HEAD                             # push current branch without typing its name
git push --force-with-lease --force-if-includes  # git >=2.30: refuses even if auto-fetch already saw the remote move
git fetch origin main:main                       # fast-forward local main without leaving your branch
git pull --rebase --autostash                    # sync with a dirty working tree
```
