# Committing — Staging Surgery, Messages, and Stash

The index is a third state between working tree and history. Fluency here is what makes small reviewable commits cheap instead of heroic.

## Staging Surgery

- `git add -p` splits a mixed working tree into clean commits: `y`/`n` per hunk, `s` to split a hunk further, `e` to edit the patch by hand (delete a `+` line, or turn a `-` line into a context line by replacing `-` with a space).
- `git add -p` cannot see a file Git does not track yet. `git add -N <file>` (intent-to-add) registers it as empty, and the next `-p` offers its contents hunk by hunk — the missing step behind "why can't I split my new file".
- `git restore -p` is the inverse: discard selected hunks from the working tree, keep the rest.
- `git stash push --staged` (git >=2.35) shelves exactly what you staged — the clean way to park half a change and test the other half.
- `git diff` shows unstaged; `git diff --staged` shows what a commit would contain. Reviewing the second one before every commit catches the debug print you forgot (Output Gate 6 in SKILL.md).
- `git commit -a` stages tracked modifications only — new files stay out. A commit that "lost" a file is usually this.

## Splitting a Commit Already Made

```bash
git rebase -i HEAD~3        # mark the offender `edit`
git reset HEAD^             # commit's changes return to the working tree, HEAD moves back
git add -p && git commit    # as many commits as it should have been
git rebase --continue
```

Verify nothing changed in aggregate: `git diff <original-sha> HEAD` must be empty.

## Message Craft

- Subject line: imperative mood, no trailing period, no ticket ID unless the repo does that. Length rule and its origin → SKILL.md rule 6 (`subject_max`, default 72 because `git log` indents bodies by four spaces inside an 80-column terminal).
- Body answers why: what was broken, what alternative was rejected, what the reader must not "fix" later. The diff already says what changed.
- Blank line between subject and body is structural, not cosmetic — without it, every tool treats the whole message as one subject.
- Trailers go last, one per line, machine-parsed: `Co-authored-by: Name <email>` (credits a pair partner on the host's contribution graph), `Signed-off-by:` via `git commit -s` (required by DCO projects), `Refs:`/`Fixes:` where the tracker consumes them. `git interpret-trailers --trailer` adds them scriptably.
- `git commit --fixup=<sha>` marks a correction; `--fixup=reword:<sha>` and `--fixup=amend:<sha>` (git >=2.32) let you fix a message or fold content while keeping the original message. All three are folded by `rebase -i --autosquash`.
- WIP commits are fine while the branch is private, and must not survive the cleanup pass — they are the reason rule 1 (rewrite only unpushed history) is worth its discipline.

## Amend

- `--amend` with an empty index edits only the message; with staged changes it folds them in. Check `git status` first so you know which of the two you are about to do.
- Amend rewrites the SHA — safe only while the commit shows in `git log @{u}..`.
- `--amend --no-edit` keeps the message; `--amend --reset-author` fixes a commit made under the wrong identity (`config.md`).
- Author date vs committer date: rebasing and amending preserve the AUTHOR date and reset the COMMITTER date. Hosts sort by author date, so a rebased branch can look stale in the UI while being freshly rebuilt; `git log --format='%h %ad %cd' --date=short` shows both.

## Empty and Mechanical Commits

- `git commit --allow-empty -m "trigger CI"` re-runs a pipeline without touching content — cleaner than a whitespace edit that pollutes blame.
- Mass reformatting goes in its own commit, alone, and its SHA goes into `.git-blame-ignore-revs` in the same change (`config.md`) — otherwise it poisons blame permanently.
- Generated files (lockfiles, snapshots) belong in the commit that caused them, never batched together at the end of the week: a lockfile commit with no source change is unreviewable.

## Stash

- A stash is a real commit object with two parents (worktree + index states), living on `refs/stash`. That is why `git stash apply <sha>` can resurrect a dropped one (SKILL.md Recovery Playbook).
- `git stash -u` includes untracked, `-a` also ignored files. Plain `stash` skips both with no warning — the most common stash data loss.
- `git stash push -m "msg" -- path/` shelves selected paths only; `git stash list` then shows meaningful names instead of "WIP on main".
- `git stash branch <name>` restores the stash onto a new branch created at the commit the stash was made from — the correct move when the stash no longer applies because the base moved.
- `pop` = `apply` + `drop`, and the drop only happens on a clean apply. After a conflicted pop, the entry is still there: resolve, then `git stash drop`.
- Stash ignores submodule state entirely: a stashed superproject leaves the submodule at whatever commit it was on (`submodules.md`).
- Long-lived stashes are lost work waiting to happen — nothing lists them at clone time and reflog expiry eventually reaches unreferenced ones. Anything worth keeping overnight belongs on a branch.

## Rewriting Your Own Branch Before Review

```bash
git rebase -i @{u}          # exactly the unpushed commits — the safe window from rule 1
```

- `pick` keep · `reword` message only · `edit` stop to change content · `squash` fold and merge messages · `fixup` fold and discard message · `drop` delete.
- Reorder lines to reorder commits; a reorder that touches the same lines conflicts, and that conflict is information: those commits were never independent.
- `git rebase --abort` returns to the start; `--skip` deletes the current commit without reporting it (`branching.md`).
