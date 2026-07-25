# High-Leverage Commands

Basics (`add`, `commit`, `push`, `pull`, `status`, `log`) are assumed. These are the flags and configs that separate fluent Git from muscle memory. History search and blame live in `forensics.md`.

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
git config --global rebase.missingCommitsCheck error   # refuse a rebase todo list that drops commits without warning
```

## Surgical State Manipulation

```bash
git add -N <file>                     # intent-to-add: makes an untracked file visible to `add -p`
git add -p                            # stage hunks, not files — split mixed work into clean commits
git restore -p                        # discard hunks selectively
git restore --source HEAD~3 -- path   # one file as it was 3 commits ago; nothing else moves
git commit --fixup <sha>              # mark a correction for a specific commit; autosquash folds it in
git stash push -m "msg" -- path/      # stash only some paths
git stash -u                          # include untracked — plain stash skips them with no warning
git cherry-pick -x <sha>              # record the origin sha in the message — traceable across branches
git rebase --onto main old-base feature   # move a branch off an abandoned or merged base
git rebase --exec 'make test' main    # run a command after every replayed commit
```

## Answering "What Is Going On"

```bash
git status -sb                        # short status + ahead/behind against upstream
git config --show-origin --get <key>  # which config file is responsible for a setting
git check-ignore -v <path>            # the exact ignore rule hiding your file
git ls-remote --heads origin          # server-side refs without cloning or fetching
git count-objects -vH                 # repo size, loose objects, pack count
git worktree list                     # every checkout backed by this repo
git describe --tags --dirty --always  # human-readable build stamp for the current commit
```

## Cloning Big Repos

```bash
git clone --filter=blob:none <url>    # blobless: full history metadata, file contents fetched on demand
git clone --depth 1 <url>             # shallow: fastest, but log/bisect/blame are truncated
```

The invisible distinction: blobless keeps every history command working (first access to old files is slower); shallow breaks them. CI throwaway checkout → shallow; a repo you will actually work in → blobless. Full strategy table, sparse checkout, and LFS: `large-repos.md`.

## Push/Pull Precision

```bash
git push origin HEAD                             # push current branch without typing its name
git push --force-with-lease --force-if-includes  # git >=2.30: refuses even if auto-fetch already saw the remote move
git push origin HEAD:main                        # push current branch to a differently named remote branch
git push --follow-tags                           # send annotated tags reachable from what you're pushing
git fetch origin main:main                       # fast-forward local main without leaving your branch
git pull --rebase --autostash                    # sync with a dirty working tree
```

## One-Off Overrides And Tracing

```bash
GIT_TRACE=1 git <cmd>                 # every subprocess and hook Git invokes — the hang debugger
git -c user.email=me@work.com commit  # one setting, one command, no config file touched
git -C ../other-repo status -sb       # run somewhere else without cd
```
