# Large Repositories — Clone Strategy, Speed, and Huge Files

Two different problems wear the same clothes: **too much history** (slow clone, slow fetch) and **too many files in the tree** (slow status, slow checkout). The fixes are different; diagnose before optimizing.

## Clone Strategies

| Strategy | Command | What still works | What breaks |
|---|---|---|---|
| Full | `git clone <url>` | Everything | Time and disk on a large repo |
| Blobless | `git clone --filter=blob:none <url>` | log, blame, bisect, diff (file contents fetched on demand, once each) | Offline work on old revisions; each first access needs the network |
| Treeless | `git clone --filter=tree:0 <url>` | Commit-graph traversal, `log --oneline` | Most path-scoped history commands refetch heavily — build-only use |
| Shallow | `git clone --depth 1 <url>` | Building HEAD | log, blame, bisect, `describe`, and any tool that reads history |
| Sparse | `--filter=blob:none --sparse` + `sparse-checkout set` | Everything, on a subset of paths | Files outside the cone simply are not on disk |

The invisible distinction: blobless keeps every history command correct; shallow truncates them without saying so. CI throwaway checkout → shallow. A repo you will actually work in → blobless. A CI job running release tooling that needs tags (`git describe`, semantic-release) → full depth, because shallow is exactly what breaks it.

Repair a shallow clone in place: `git fetch --unshallow`.

## Sparse Checkout

```bash
git clone --filter=blob:none --sparse <url> && cd repo
git sparse-checkout set packages/app packages/shared    # cone mode: whole directories
git sparse-checkout add packages/tools
git sparse-checkout list
git sparse-checkout disable                              # back to the full tree
```

- Cone mode matches directory prefixes and stays fast; it is the default for `sparse-checkout set` on git >=2.37 and needs an explicit `init --cone` before that. Non-cone mode accepts full gitignore-style patterns and degrades as the pattern list grows — reach for it only when directory granularity genuinely cannot express the subset.
- Files outside the cone still exist in history and in commits you make; a tool that writes to an excluded path produces changes you cannot see in `git status`.
- Sparse index (`git sparse-checkout init --sparse-index`) shrinks the index itself, which is what makes `status` fast on a monorepo — without it you still pay for every path in the repo.

## Making Everyday Commands Fast

- `git maintenance start` (git >=2.30) registers background jobs: prefetch, incremental repack, commit-graph, loose-object cleanup. It replaces hand-rolled `gc` cron jobs and keeps fetch times flat.
- `git config core.fsmonitor true` (builtin since git >=2.37) plus `git config core.untrackedCache true` removes the full-tree stat sweep that dominates `git status` on trees with hundreds of thousands of files.
- `git config feature.manyFiles true` bundles the index-side settings (index version 4, untracked cache) in one switch.
- The commit-graph file is what makes `git log --graph`, `merge-base`, and ahead/behind counts cheap; `git commit-graph write --reachable` builds it once, maintenance keeps it current.
- Automatic gc triggers on loose-object pressure (`gc.auto`, 6700 loose objects by default) and pack count (`gc.autoPackLimit`, 50 packs). `git count-objects -vH` shows both, and explains a repo that pauses to "auto pack" at inconvenient moments.

## Finding What Made The Repo Huge

```bash
git count-objects -vH                       # size-pack is the number that matters
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob"' | sort -k3 -n | tail -20
```

The tail is your list of offenders with their paths. A repo is rarely big because of code; it is big because of one committed dataset, a vendored binary, or a video from 2019.

## Git LFS

- LFS replaces the file in the tree with a small pointer and stores the bytes on a separate server, wired in through `.gitattributes` (`*.psd filter=lfs diff=lfs merge=lfs -text`).
- **Adding LFS today does not shrink yesterday's history.** Existing blobs stay in the pack files. Rewriting is the only way down: `git lfs migrate import --above=10MB --everything` — same blast radius as any history rewrite (SKILL.md rule 8), so treat it like `secrets.md`: coordinate, then everyone re-clones.
- `GIT_LFS_SKIP_SMUDGE=1 git clone <url>` clones pointers only; `git lfs pull --include="assets/current/**"` then fetches just what you need. This is the fix for a CI checkout that spends most of its wall time downloading assets the job never opens.
- LFS costs are storage AND bandwidth quotas on hosted plans; a CI job that re-downloads assets on every run is the usual reason a quota evaporates. Cache the LFS objects in CI.
- Anything that reads the repo without the LFS filter installed (a plain `git archive`, some deployment tooling, an unconfigured runner) gets pointer text files instead of content, and the failure looks like corrupted assets.

## Hosting Limits And Monorepo Etiquette

- GitHub warns >50 MB, blocks >100 MB per file (SKILL.md Traps), and recommends keeping repositories under a few GB — pushes that hit the limit fail server-side, after your upload (`errors.md`). Those are GitHub's numbers; every other `remote_host` sets its own, so read the limit before sizing a guard around it.
- Path-scoped history stays cheap if you avoid repo-wide reformat commits; each one touches every file and inflates every subsequent `log -- path` walk.
- CI in a monorepo should fetch with a filter and check out a cone; a full clone per job multiplied by a hundred jobs a day is the single largest CI cost most teams never look at.
- Splitting a monorepo later is mechanical (`git filter-repo --path`, `remotes.md`); un-splitting is not. That asymmetry, not aesthetics, is the honest argument for starting monorepo.
