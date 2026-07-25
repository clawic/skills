# Tags, Releases, and Backports

## Annotated vs Lightweight

- Annotated (`git tag -a v1.2.0 -m "..."`) creates a real object with tagger, date, message, and optional signature. Lightweight (`git tag v1.2.0`) is just a ref pointing at a commit.
- Rule: annotated for anything anyone else will see. `git describe` only considers annotated tags by default, and a lightweight tag carries no record of who cut the release or when.
- `git show v1.2.0` on an annotated tag prints the tag object first; scripts that expect a commit need the peeled form `v1.2.0^{}` (SKILL.md Revision Syntax).
- Sign releases where consumers verify them: `git tag -s`, checked with `git tag -v` (`config.md`).

## Pushing And Fetching Tags

- Tags do not travel with a normal push. `git push origin v1.2.0` sends one; `git push --follow-tags` sends the annotated tags reachable from the commits you are already pushing — the sane default for a release branch.
- `git push --tags` sends every local tag including experiments and lightweight scratch tags. Rarely what you want.
- **Moving a published tag is the cardinal sin.** Clients that already have the tag keep the old one: `git fetch` does not update an existing tag ref without `--force`. Half your team, and every cached CI checkout, then builds a different commit under the same version string. Fix forward with a new version number instead.
- Deleting: `git tag -d v1.2.0` locally, `git push origin --delete v1.2.0` remotely. Everyone who already fetched it keeps it, and hosted release artifacts usually survive the tag.
- `git fetch --tags --force` is the repair command when tags have diverged.

## Version Stamps In Builds

- `git describe --tags --dirty --always` yields `v1.2.0-14-gabc1234[-dirty]`: nearest tag, commits since, short SHA, plus a marker for uncommitted changes. It is the cheapest honest build identifier.
- It needs tags in the clone. Shallow CI checkouts have none, which is why release tooling fails with "no names found" — set the checkout to full depth for jobs that version or publish (`large-repos.md`).
- Monorepos prefix tags per package (`pkg-a/v1.2.0`) and scope the lookup: `git describe --tags --match 'pkg-a/*'`.
- `git tag --sort=-v:refname` sorts by version semantics, not lexically — without it `v1.10.0` sorts before `v1.9.0`.

## Cutting A Release

1. `git log --no-merges --pretty='%s (%an)' v1.1.0..HEAD` — the raw changelog material; `--first-parent` instead if the repo merges every PR.
2. Tag the exact commit that was tested, usually the merge commit on the release branch — not "the tip of main right now".
3. Push commit and tag together (`--follow-tags`) so CI never sees a tag pointing at an unpushed commit.
4. Record the tag → SHA mapping wherever deployments are tracked; a tag is a movable name, a SHA is the artifact identity.

Under a squash-merge policy the PR title — the MR title where `remote_host` is gitlab — becomes the only surviving commit message, so it is also the only changelog input; that is what makes title discipline a release concern (`collaboration.md`).

## Release Branches And Backports

- Create from the tag, not from main: `git switch -c release/1.2 v1.2.0`. Starting from main pulls in everything merged since, with no warning.
- Backport with `git cherry-pick -x <sha>` — `-x` appends "(cherry picked from commit ...)", which is how anyone later proves the fix reached both lines.
- Find what has not been backported: `git cherry -v release/1.2 main` lists main's commits by patch-id and marks with `+` those absent from the release branch. Renames and context drift defeat patch-id matching, so treat `+` as a candidate list, not a verdict.
- A cherry-pick that conflicts every time on the same file is a signal the branches have structurally diverged: port the fix as a fresh commit written against the release branch instead of forcing the patch.
- Merging a release branch back into main duplicates commits under new SHAs unless it was branched from main and never rewritten — usually the cleaner path is to cherry-pick forward and never merge backwards.

## Hotfix Without A Release Branch

```bash
git switch -c hotfix/1.2.1 v1.2.0     # detached tag becomes a branch
# fix, test, commit
git tag -a v1.2.1 -m "Fix X"
git push origin hotfix/1.2.1 --follow-tags
git switch main && git cherry-pick -x <fix-sha>    # main must get it too
```

The step everyone forgets is the last one: a hotfix that never lands on main returns as a regression in the next release.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Retagging after a "small fix" | Existing clones and caches keep the old tag; two artifacts share a version | Bump the patch number; versions are cheap |
| Tagging the branch tip instead of the tested commit | The tip may include a merge that landed during the release | Tag the SHA that CI validated |
| Relying on `git describe` in a `--depth 1` CI clone | No tags fetched, no names found | Full-depth checkout for release jobs |
| Changelog generated from raw `git log` on a squash-merge repo | Every entry is a PR title of unknown quality | Enforce PR-title standards, or generate from labeled PRs |
| Deleting a bad tag and reusing the name minutes later | Whoever fetched in between is permanently on the old commit | New name, always |
