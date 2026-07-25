# Remotes, Forks, and Moving Repositories

A remote is a name bound to a URL plus a refspec. Everything confusing about remotes is a refspec you never looked at.

## Inspect And Repoint

```bash
git remote -v                                   # URLs per remote, fetch and push separately
git remote show origin                          # tracking branches, stale refs, what push would do
git remote set-url origin git@host:org/repo.git # switch protocol or host without re-cloning
git remote add upstream https://host/org/repo.git
git ls-remote --heads --tags origin             # refs on the server, without cloning anything
git config --get remote.origin.fetch            # the refspec that decides what fetch brings back
```

`git ls-remote` answers "does that branch/tag exist yet" and "which SHA is main" in one round trip — the check to run before scripting anything against a remote.

## Fork Workflow

- Convention: `origin` = your fork, `upstream` = the source. Keep it that way; every guide and every teammate assumes it.
- Sync: `git fetch upstream && git rebase upstream/main` on your topic branch. Rebasing keeps your PR reviewable as your own commits; merging upstream into the branch buries them under merge commits.
- **Never open a PR from your fork's `main`.** Maintainers who push a fixup land it on your default branch, your next sync conflicts with your own PR, and you cannot open a second PR without waiting. Always a topic branch.
- Contributions from a fork run CI with reduced permissions on most hosts — a workflow that needs secrets will fail for outside contributors by design, not by misconfiguration.

## Refspecs In Practice

- Default fetch refspec: `+refs/heads/*:refs/remotes/origin/*`. The `+` means non-fast-forward updates are allowed for remote-tracking refs — which is why a rebased upstream branch updates cleanly while your local branch does not.
- `git fetch origin main:main` fast-forwards your local `main` without checking it out. It fails (correctly) if the update would not be a fast-forward.
- `git push origin HEAD:main` pushes the current branch to a differently named remote branch — the move for "my local branch name is wrong but the work is right".
- `git push origin :old-branch` deletes a remote branch (empty source). `git push origin --delete old-branch` says the same thing readably.
- Fetch only what you need from a huge remote: `git fetch --depth 1 origin <sha>` retrieves a single commit's tree, useful for CI that only builds one revision.

## Keeping Refs Clean

- `git fetch --prune`, or `git config --global fetch.prune true`, deletes remote-tracking refs for branches removed on the server. Without it, `git branch -r` grows forever and tab-completion becomes useless.
- Pruning the tracking ref does not delete your local branch: `git branch -vv | grep ': gone]'` lists local branches whose upstream disappeared — the safe input to a cleanup pass.
- `git remote prune origin --dry-run` shows what would go before it goes.

## Multiple And Mirrored Remotes

- Push to two places at once by adding a second push URL: `git remote set-url --add --push origin git@backup:org/repo.git` (add the original again first — the first `--add --push` replaces the implicit default).
- `git clone --mirror` copies EVERY ref, including `refs/pull/*` on GitHub and any internal refs. Pushing that to a new host imports thousands of pull-request refs you did not ask for. For a clean migration, mirror-clone, then push only what you want:
  ```bash
  git clone --mirror git@old:org/repo.git && cd repo.git
  git push --prune git@new:org/repo.git 'refs/heads/*:refs/heads/*' 'refs/tags/*:refs/tags/*'
  ```
- After migrating, tell collaborators to `git remote set-url origin <new>`; a stale origin means work keeps landing on the abandoned host with nothing to signal it.

## Offline And Air-Gapped Transfer

```bash
git bundle create repo.bundle --all          # every ref in one file
git bundle verify repo.bundle                # check prerequisites before trusting it
git clone repo.bundle repo                   # a bundle is a valid clone source
git bundle create update.bundle main ^v1.2.0 # incremental: only what's new since the tag
```

Bundles are also the simplest full backup of a repo that lives only on one machine (`recovery.md`).

## Splitting And Combining Repositories

```bash
# extract a subdirectory into its own repo, history intact
git clone <url> extracted && cd extracted
git filter-repo --path packages/x --path-rename packages/x/:

# fold a repo into a monorepo subdirectory
# → submodules.md, "Merging Whole Repositories"
```

Both operations rewrite every SHA, so treat them like any history rewrite: announce, re-clone, and update pinned references (`secrets.md`, "After Any History Rewrite").

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Pushing to a non-bare repo on a server | `receive.denyCurrentBranch` refuses to update a checked-out branch | Push to a bare repo, or deploy by pulling |
| Assuming `git remote remove` deletes anything remote | It only forgets a local name and its tracking refs | Delete the repository on the host if that is the intent |
| Fetching a rewritten upstream and merging | The merge resurrects the pre-rewrite objects for everyone | `git reset --hard upstream/main` after an announced rewrite |
| Adding a token to the remote URL for CI | It persists in `.git/config` and in any pasted `git remote -v` | Credential helper or SSH deploy key (`config.md`) |
| Cloning with `--mirror` to "make a backup you can work in" | A mirror is bare and its refs are managed as exact copies; committing in it is not the workflow you expect | `git bundle` for backups, a normal clone for work |
