# Committed Secrets and Unwanted Blobs

## The First Ten Minutes

Order matters, and the order is not negotiable:

1. **Rotate the credential.** Assume it is compromised the moment it left your machine. Every later step is cleanup, not containment (SKILL.md rule 7).
2. **Find the blast radius.** Was it pushed? To a public repo? Is there a fork, a mirror, a CI cache, a build artifact, a Slack paste of the diff?
3. **Check the provider's logs** for use of the credential between commit and rotation. That is the question an incident review will ask.
4. Only then decide whether to rewrite history.

A secret in an unpushed commit is a local problem: `git reset --soft`, remove it, recommit. A pushed secret is an incident.

## Why Rewriting Is Not Containment

- Anyone who cloned or forked keeps the object. On GitHub, a commit pushed to a fork network stays reachable by SHA from the upstream repository even after the branch is deleted — until the host garbage-collects it, which you cannot trigger yourself. Support can expunge it; the credential is still burned.
- CI caches, artifact stores, container layers built from that checkout, and `pip`/`npm` caches all hold copies.
- Search engines and secret-scanning bots index public repos within minutes. Public leak → the credential was used before you finished reading this file.

Rewrite anyway when the repo is private and the audience is small, or when the file must not persist for compliance reasons. Do not let the rewrite delay the rotation.

## Removing It From History

```bash
git clone --mirror git@host:org/repo.git repo-backup.git   # non-negotiable safety copy
cd repo
git filter-repo --invert-paths --path config/secrets.yml   # remove a path everywhere
git filter-repo --replace-text expressions.txt             # or redact strings, keeping the file
```

`expressions.txt` holds one rule per line: `literal:AKIAIOSFODNN7EXAMPLE==>REDACTED` or `regex:ghp_[A-Za-z0-9]{36}==>REDACTED`.

- Use `git filter-repo`, not `filter-branch`: `filter-branch` is deprecated by Git's own documentation, orders of magnitude slower, and mishandles tags and refs in ways that produce a half-rewritten history. BFG Repo-Cleaner is a valid alternative for the simple "delete these files / replace these strings" case.
- `filter-repo` removes the `origin` remote and expires reflogs on purpose, so you cannot accidentally keep both histories or push a mixed state. That is also why the mirror backup above is mandatory (`recovery.md`).
- Push the rewrite: `git push --force --all && git push --force --tags`. Open PRs break, every clone must be re-cloned or hard-reset, and CI will rebuild everything.
- Coordinate: announce it, and check afterwards that nobody merged a branch based on the old history — that single merge reintroduces every removed object.

## Stopping The Next One

- `.gitignore` the real config file, commit a `config.example` with placeholder values. Never `--assume-unchanged` a tracked secret file: Git keeps tracking it and one `git stash`/`checkout` republishes your edits (SKILL.md Traps).
- A `pre-commit` secret scanner is the cheapest control available; a `pre-push` one catches what `--no-verify` skipped (`hooks.md`). Host-side push protection is the only layer that cannot be bypassed locally.
- Scan history once, when you adopt scanning: existing repos usually contain something, and finding it during a quiet afternoon beats finding it during an audit.
- Keep secrets out of the argv and the environment of committed scripts too — `git log -p` of a CI script that echoes a token leaks exactly as well as a config file.
- `git diff --cached` before every commit (Output Gate 6 in SKILL.md) prevents most of these without any tooling at all.

## Unwanted Blobs (the same mechanics, lower stakes)

- A 400 MB dataset committed once inflates every clone forever, even after deletion — the object stays in the pack.
- Locate the offenders with the object-size scan in `large-repos.md`, then remove by path (`filter-repo --invert-paths --path data/dump.sql`) or by size (`filter-repo --strip-blobs-bigger-than 10M`).
- If the file is legitimately needed, migrate it to LFS instead of deleting it (`large-repos.md`) — same rewrite, different destination.
- Verify the shrink: `git count-objects -vH` before and after, and confirm the new pack size actually dropped before asking colleagues to re-clone.

## After Any History Rewrite

- Every collaborator: `git fetch --all && git reset --hard origin/<branch>`, or a fresh clone. A `git pull` on the old history creates a merge that resurrects the removed objects.
- Re-cut any release tags that pointed at rewritten commits, and tell whoever consumes them (`releases.md`).
- Update pinned SHAs elsewhere: submodule gitlinks, deployment manifests, changelog links, issue references. All of them now point at commits that no longer exist.
