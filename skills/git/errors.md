# Errors — Message to Cause in One Step

Git's messages name the mechanism, not the fix. Match the message, apply the check; each entry is ordered by how often it is the real cause.

## Refuses To Start

**`fatal: Unable to create '.../.git/index.lock': File exists`**
Another Git process holds the index, or one crashed. Check for a live process (an IDE fetching, a hook, a paused rebase) before deleting anything. Only when nothing is running: `rm -f .git/index.lock`. Deleting the lock under a live process corrupts the index.

**`fatal: not a git repository (or any of the parent directories)`**
Wrong directory, or the `.git` file of a linked worktree points at a main repo that moved (`worktrees.md`). `git rev-parse --show-toplevel` confirms where Git thinks it is.

**`fatal: detected dubious ownership in repository at ...`**
`git >=2.35.2` refuses repos owned by another user — normal in containers, CI, and after `sudo`. Fix the ownership if you can (`chown -R`); otherwise `git config --global --add safe.directory /path`. `safe.directory '*'` disables the check entirely and is a container-only shortcut, never a workstation default.

## Refuses To Change The Working Tree

**`error: Your local changes to the following files would be overwritten by checkout/merge`**
The incoming version touches files you edited. Options, in order: commit them, `git stash -u`, or `git switch -m <branch>` to carry them across with a merge.

**`error: The following untracked working tree files would be overwritten`**
The branch you are entering contains a file you have untracked on disk. Rename or delete your copy — Git will not clobber it without asking.

**`fatal: pathspec 'x' did not match any files`**
Three causes: a typo/wrong cwd; the file is ignored (`git check-ignore -v x` names the rule); or Git is reading the argument as a revision — separate them with `--` (`git checkout -- x`).

**`fatal: cannot lock ref 'refs/heads/feature': 'refs/heads/feature/x' exists`**
Refs are files: a branch named `feature/x` creates a directory `feature/`, so a branch named plain `feature` can no longer exist. Pick flat or hierarchical naming, not both (`branching.md`).

## Refuses To Integrate

**`hint: You have divergent branches and need to specify how to reconcile them`**
`git pull` with no strategy configured. `git >=2.27` warns and newer versions refuse outright. Set it once: `git config --global pull.rebase true` (`commands.md`).

**`fatal: refusing to merge unrelated histories`**
The two branches have different root commits — a `git init` that later got a remote added, a shallow clone, or an accidental second repo. `--allow-unrelated-histories` is correct only when you actually intend to graft two projects together (`remotes.md`); otherwise you are merging the wrong thing.

**`error: cannot pull with rebase: You have unstaged changes`**
`git pull --rebase --autostash`, or set `rebase.autostash true` permanently.

**`CONFLICT (modify/delete)` / `CONFLICT (rename/rename)`**
Not a plain content conflict: one side deleted or moved the file. Decide the file's fate first (`git rm` or `git add`), then the content — three-way tools are useless here (`conflicts.md`).

## Refuses To Push

**`! [rejected] main -> main (fetch first / non-fast-forward)`**
The remote moved. `git pull --rebase` then push. This message never means "you need force" — forcing here overwrites a teammate's commits.

**`! [rejected] (stale info)`**
`--force-with-lease` did its job: the remote changed since your last fetch. Fetch, inspect `git log @{u}..` and `git log ..@{u}`, then decide (`collaboration.md`).

**`! [remote rejected] (pre-receive hook declined)`**
Server-side policy: branch protection, required checks, push protection on a detected secret (`secrets.md`), a blocked file size, or a required signature. The hook usually prints the reason on the line above — read it before retrying.

**`fatal: The current branch X has no upstream branch`**
`git push -u origin HEAD` once, or `push.autoSetupRemote` (git >=2.37) forever.

**`remote: error: refusing to update checked out branch`** (`receive.denyCurrentBranch`)
You pushed to a non-bare repository whose working tree is on that branch. Push to a bare repo, or push to a different branch and merge there.

**`error: RPC failed; curl 92 HTTP/2 stream ... / HTTP 413`**
The push body exceeded a server or proxy limit — usually one enormous blob, sometimes an entire history pushed at once. Split the push (`git push origin <sha>:refs/heads/main` in chunks), or move the blob to LFS (`large-repos.md`). Raising `http.postBuffer` helps only when a single object exceeds the buffer; it is cargo cult in every other case.

## Refuses To Authenticate

**`Permission denied (publickey)`**
`ssh -T git@<host>` isolates Git from SSH. Then: is the key loaded (`ssh-add -l`), is the right key offered for this host (`~/.ssh/config` Host block), and is the key registered on the account? Multiple accounts on one host need distinct Host aliases (`config.md`).

**`fatal: could not read Username for 'https://...': terminal prompts disabled`**
An HTTPS remote with no credential helper in a non-interactive context (CI, a hook, an agent). Use SSH, a token in the remote URL's credential store, or `GIT_TERMINAL_PROMPT=0` plus a helper configured up front.

**`remote: Support for password authentication was removed`**
The host wants a token or SSH key. If it "used to work", a cached old credential is being replayed: clear it (`git credential-osxkeychain erase`, or the platform's helper) before storing the new one (`config.md`).

## Confusing But Not Errors

**`You are in 'detached HEAD' state`**
Normal after checking out a tag, a SHA, or during rebase and bisect. Commits made here belong to no branch. Keep them with `git switch -c <name>`; leave without keeping them by switching branches (they stay reflog-reachable for 30 days).

**`warning: LF will be replaced by CRLF`**
`core.autocrlf` is rewriting line endings on checkout. Harmless once, a repo-wide diff storm when configured inconsistently across machines — fix it in `.gitattributes` (`config.md`).

**`Auto packing the repository for optimum performance`**
Automatic `git gc` fired on loose-object pressure. Never interrupt it; if it is frequent, the repo needs maintenance (`large-repos.md`).

**Nothing at all happens** — the command hangs with no output: a hook waiting on stdin, a credential prompt swallowed by a non-TTY, or an editor that never opened. `GIT_TRACE=1 git <cmd>` prints each step and names the culprit.
