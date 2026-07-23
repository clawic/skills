---
name: git
slug: git
version: 1.0.10
description: Git commits, branches, rebases, merges, conflict resolution, history recovery, and team workflows for safe version control. Use when (1) the task touches Git, a repository, commits, branches, merges, rebases, or pull requests; (2) history safety, collaboration, or recovery matter; (3) the agent should automatically apply Git discipline instead of improvising.
homepage: https://clawic.com/skills/git
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 📚
    requires:
      bins:
      - git
    os:
    - linux
    - darwin
    - win32
    displayName: Git
    configPaths:
    - ~/Clawic/data/git/
---

## When To Use

- Any task touching a Git repository: commits, branches, merges, rebases, pull requests, tags, stashes.
- Something looks lost — deleted branch, bad rebase, hard reset, vanished stash: recovery is a first-class use (→ Recovery Playbook).
- Before any history rewrite or force push: run the Output Gates below first.
- Not for GitLab pipelines or merge-request settings — that is the `gitlab` skill.

Stateless: apply by default whenever Git work is part of the job. If the user shares workflow preferences, they live in `~/Clawic/data/git/memory.md` (→ `setup.md`).

## Quick Reference

| Situation | File |
|-----------|------|
| Need a high-leverage flag or one-time config | `commands.md` |
| Interactive rebase, bisect, worktree, sparse checkout, subtree | `advanced.md` |
| Branch creation, switching, or merging going wrong | `branching.md` |
| Merge or rebase stopped on conflicts | `conflicts.md` |
| Rewriting, recovering, or investigating history | `history.md` |
| Push/pull/review friction with a team | `collaboration.md` |
| User states workflow preferences | `setup.md` + `memory-template.md` |
| Anything else | Stay here — this file covers the default path |

## Core Rules

1. **Rewrite only unpushed history.** Check: `git log @{u}..` lists exactly the commits safe to amend/rebase/squash. A commit missing from that output is published — fix forward with `git revert` instead.
2. **Force push = `--force-with-lease`, own branch only, never main.** The lease is void if anything fetched after the remote moved — IDE auto-fetch does this silently — so add `--force-if-includes` (git >=2.30) to close that hole.
3. **Destructive command → name the casualties first.** `git status` before `reset --hard`, `git clean -n` before `git clean -f`, `git diff @{u}` before force push. If you cannot list what dies, you are not ready to run it.
4. **`git reflog` before panic.** Committed work survives 90 days (reachable) / 30 days (unreachable). The only truly unrecoverable losses: never-staged edits killed by `reset --hard`/`restore`, and untracked files killed by `clean -f`.
5. **Conflicts repeat per commit in rebase, once in merge.** Same region conflicting across 3+ replayed commits → abort and merge instead (one resolution), or enable `rerere` so each resolution is recorded once.
6. **Commit = one reviewable change.** Mixed changes → `git add -p` to split. Subject: aim ≤50 chars (convention), hard stop 72 — GitHub truncates longer subjects. Body explains why, not what.
7. **A pushed secret is a leaked secret.** Rotate the credential first; history rewrite (filter-repo) is cleanup, not containment — forks, clones, and CI caches keep the old objects.

## Recovery Playbook

| Symptom | Play |
|---------|------|
| Just finished a bad rebase/merge/reset | `git reset --hard ORIG_HEAD` — Git saved your pre-operation position |
| Commits vanished after a rewrite | `git reflog` → `git reset --hard HEAD@{n}` (the entry just before the damage) |
| Deleted a branch | `git reflog` → `git branch restored <sha>` |
| Committed on the wrong branch (unpushed) | `git switch right-branch && git cherry-pick <sha>`, then `git switch - && git reset --hard HEAD~1` |
| Committed on detached HEAD, then switched away | Commit stays in `git reflog` for 30 days → `git branch rescue <sha>` |
| Need one file as it was N commits ago | `git restore --source HEAD~N -- path` — nothing else moves |
| `reset --hard` ate staged-but-uncommitted work | `git fsck --lost-found` — staged content survives as dangling blobs; never-staged edits are gone |
| Dropped a stash | `git fsck --unreachable \| grep commit` → `git stash apply <sha>` |
| Bad commit already on a shared branch | `git revert <sha>` — never rewrite shared history |
| Unsure what happened / anything else | `git reflog` first; before experimenting, `git branch backup` — a branch is free insurance |

## Commit Discipline

- One reviewable change per commit; `git add -p` splits mixed work. `git commit --fixup <sha>` + `git rebase -i --autosquash` batches corrections without "fix typo" noise.
- Match the repo's existing message style before imposing one — check `git log --oneline -20`. Conventional commits (`type(scope): description`) only where the log or release tooling already uses them.
- `git pull --rebase --autostash` before push: no surprise merge commits, works with a dirty tree.
- First push: `git push -u origin HEAD`, or set `push.autoSetupRemote` (git >=2.37) once and never type `-u` again.

## Conflict Basics

- Set `merge.conflictStyle zdiff3` (git >=2.35) once: markers include the base version, so you see what each side changed instead of guessing intent.
- Ours/theirs invert during rebase: "ours" = the branch you are rebasing onto, "theirs" = your own commit being replayed — rebase checks out upstream and applies your commits as patches on top.
- After resolving: `git grep -nE '^(<{7}|={7}|>{7})'` must return nothing, then build/tests, then `--continue`.
- Everything else → `conflicts.md`.

## Bisect

- Cost formula: steps = ⌈log2(commits in range)⌉ — 1,000 commits ≈ 10 tests, so bisect beats reading diffs on anything but a trivial range.
- With a test command it runs unattended: `git bisect start HEAD v1.0.0 && git bisect run ./test.sh` (exit 0 = good, 1–127 = bad, 125 = skip this commit).
- Commit won't build → `git bisect skip`; finished → `git bisect reset`.

## Output Gates

Before declaring Git work done:

- [ ] Rewrite gate: every amended/rebased commit appeared in `git log @{u}..` before I touched it
- [ ] Deletion gate: I listed the casualties (`git status`, `git clean -n`) before any `--hard` / `-f`
- [ ] Push gate: not forcing to a shared branch; if forcing at all, `--force-with-lease` minimum
- [ ] Conflict gate: marker grep is clean AND the project builds after resolution
- [ ] Identity gate: `git config user.email` is the right identity for this repo (work vs personal)

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| `git stash` before a risky operation, expecting untracked files saved | Plain stash skips untracked files | `git stash -u` |
| Assuming `stash pop` dropped the stash after a conflict | Pop only drops on clean apply; on conflict the entry stays | Resolve, then `git stash drop` |
| `git clean -fdx` to "clean build artifacts" | `-x` also deletes ignored files: `.env`, IDE config, local secrets | `git clean -fd`, always preview with `-n` |
| Renaming `Foo.js` → `foo.js` works locally, breaks CI | macOS/Windows filesystems are case-insensitive, Linux is not | `git mv Foo.js tmp && git mv tmp foo.js` |
| `git branch -d` refuses to delete after squash-merge | Squashed commits are unreachable from main, so Git thinks the branch is unmerged | Verify the PR merged, then `-D` |
| Committing a large binary "just this once" | GitHub warns >50 MB, blocks >100 MB — and history keeps the blob forever | Git LFS before first commit; after the fact, `git filter-repo` |
| "Sharing" hooks via `.git/hooks` | `.git/hooks` is never versioned | Commit a hooks dir + `git config core.hooksPath` |
| Cloned repo has empty submodule directories | Submodules need explicit initialization | `git clone --recurse-submodules`, or `git submodule update --init` |

## Where Experts Disagree

- **Squash-merge vs preserve commits.** Squash school treats PR commits as WIP noise (small PRs, GitHub-centric teams); preserve school needs atomic commits for bisect and surgical reverts (long-lived repos, kernel-style review). Follow the repo's existing merge style — mixing styles hurts more than either choice.
- **Trunk-based vs long-lived branches.** DORA research associates branches merged within about a day with higher delivery performance; git-flow-style release branches pay off only when you maintain multiple released versions in parallel.
- **Conventional commits.** Pay off when tooling consumes them (changelog generation, semver automation); pure ceremony otherwise. Detect: release automation in the repo → use them.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/git (install if the user confirms):
- `gitlab` — GitLab CI/CD and merge requests
- `docker` — Containerization workflows
- `code` — Code quality and best practices

## Feedback

- If useful, star it: https://clawic.com/skills/git
- Latest version: https://clawic.com/skills/git

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/git.
