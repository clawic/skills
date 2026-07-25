# Config, Identity, Auth, and Ignore Rules

## Where A Setting Comes From

Precedence, lowest to highest: system (`/etc/gitconfig`) → global (`~/.gitconfig`) → local (`.git/config`) → worktree (with `extensions.worktreeConfig`) → `-c key=value` on the command line.

```bash
git config --show-origin --get user.email     # which file set it
git config --list --show-scope                # everything, with its level
```

Debugging any "Git ignores my setting" report starts with `--show-origin`; the usual answer is a local value shadowing the global one, or a tool that wrote `.git/config` behind you.

## Identity Per Directory

```bash
# ~/.gitconfig
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
```

- The trailing slash matters: `gitdir:~/work/` matches everything under it; without it, only that exact path.
- `gitdir/i:` is the case-insensitive variant, needed on macOS and Windows.
- Deliberately leave `user.email` unset globally: Git then refuses to commit outside a configured directory instead of attributing work to the wrong identity with no warning.
- Already committed under the wrong identity: `git commit --amend --reset-author` for the last one; for a range, rebase with `--exec 'git commit --amend --reset-author --no-edit'`, and only while the commits are unpushed (SKILL.md rule 1).
- `.mailmap` fixes how names and emails DISPLAY in `git log` and `shortlog` without rewriting anything — the right tool when the history is published.

## Authentication

- SSH vs HTTPS is a per-remote choice, not a repo-wide one. `git remote -v` shows which you are on; `git remote set-url` switches.
- Rewrite protocols globally without touching each repo: `git config --global url."git@host:".insteadOf "https://host/"` — the standard fix for private submodules and vendored dependencies fetched over HTTPS in CI.
- Multiple accounts on one host need SSH `Host` aliases:
  ```
  Host github-work
      HostName github.com
      IdentityFile ~/.ssh/id_work
      IdentitiesOnly yes
  ```
  then a remote URL of `git@github-work:org/repo.git`. Without `IdentitiesOnly`, the agent offers keys in its own order and the wrong account authenticates — the cause of "pushed to the right repo as the wrong user".
- Credential helpers cache HTTPS tokens: `osxkeychain`, `libsecret`, `manager`, or `cache --timeout=3600`. A rotated token that keeps failing is the OLD one being replayed from the store — erase it (`printf 'protocol=https\nhost=github.com\n' | git credential-osxkeychain erase`) before storing the new one.
- Never put a token in the remote URL: it lands in `.git/config`, in shell history, and in any `git remote -v` output someone pastes into a ticket.

## Signing

```bash
git config --global gpg.format ssh                              # git >=2.34
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

- SSH signing reuses a key you already have; GPG remains correct where a web of trust or a hardware token policy already exists.
- Local verification needs an allowed-signers file (`gpg.ssh.allowedSignersFile`) mapping emails to public keys; without it `git log --show-signature` reports "no principal matched" even for your own valid signatures.
- A host shows "Verified" only when the key is registered as a SIGNING key on the account — registering it for authentication alone is the usual reason a correctly signed commit shows as unverified.
- Signing proves a key signed it, not that a human intended it (SKILL.md, Where Experts Disagree).

## Ignore Rules

- Precedence: the LAST matching pattern wins, and per-directory `.gitignore` files override the ones above them.
- `git check-ignore -v <path>` prints the exact file, line, and pattern responsible. Use it before arguing with the ignore file.
- Ignore rules never apply to tracked files — untracking needs `git rm --cached <path>` and a commit (SKILL.md Traps).
- A negation cannot rescue a file whose parent directory is excluded: `logs/` then `!logs/keep.txt` fails, because Git never descends into an excluded directory. Exclude the contents instead: `logs/*` then `!logs/keep.txt`.
- Personal, unshared ignores go in `.git/info/exclude` (per repo) or `core.excludesFile` (global) — editor and OS clutter does not belong in the project's `.gitignore`.
- `--assume-unchanged` and `--skip-worktree` are not "ignore for tracked files". `--assume-unchanged` is a performance promise you are breaking (Git may notice changes anyway); `--skip-worktree` makes Git preserve your local version and fail confusingly on incoming changes. The correct pattern remains: tracked `.example` template plus an ignored real file.

## Line Endings And Attributes

- Fix line endings in the repo, once, with `.gitattributes` — not per machine with `core.autocrlf`:
  ```
  * text=auto eol=lf
  *.bat text eol=crlf
  *.png binary
  ```
- After adopting it: `git add --renormalize . && git commit`. Skipping this leaves the whole repo looking modified on the next checkout for everyone on Windows.
- Custom merge drivers stop lockfile conflicts, but the driver must exist locally: `.gitattributes` `package-lock.json merge=ours` does nothing until each machine runs `git config merge.ours.driver true`. Attributes travel, driver definitions do not.
- `linguist-generated=true` collapses generated files in host diffs; `export-ignore` keeps files out of `git archive`; `diff=python`/`diff=go` gives hunk headers that name the enclosing function.
- macOS: `core.precomposeunicode true` prevents accented filenames appearing as phantom renames between macOS and Linux. Windows: `core.longpaths true` for deep dependency trees, and `core.symlinks` is off when the account lacks the privilege — symlinked files arrive as text files containing a path, which looks like corruption.
- Git tracks files, not directories: an empty directory can never be committed. `.gitkeep` is a plain file with no special meaning to Git — any tracked file in the directory works equally well.
- `core.ignorecase` is set at clone time from the filesystem — this is why a case-only rename needs the two-step `git mv` dance (SKILL.md Traps).

## Aliases Worth The Line

```bash
git config --global alias.st "status -sb"
git config --global alias.lg "log --oneline --graph --decorate"
git config --global alias.unstage "restore --staged"
git config --global alias.last "log -1 --stat"
git config --global alias.pushf "push --force-with-lease --force-if-includes"
```

An alias starting with `!` runs a shell command from the repository root, not from your current directory — the most common reason a shell alias works from the top level and fails in a subdirectory.
