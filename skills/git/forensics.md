# Forensics — When Did This Break, Who Wrote It, Where Did It Go

History is a searchable database. Most "nobody knows why this code exists" answers are one command away.

## Question → Tool

| Question | Command |
|---|---|
| Which commit introduced this string? | `git log -S "text" --oneline` (pickaxe: counts of the string changed) |
| Which commits touched code matching this pattern? | `git log -G "regex" -p` (catches moves that `-S` reports as no-op) |
| How did this function evolve? | `git log -L :funcName:path/file.ts` |
| Who last touched these lines, ignoring noise? | `git blame -w -C -C -C -L 40,80 path` |
| When did this line DISAPPEAR? | `git blame --reverse <old-sha>..HEAD -- path` |
| Which commit broke the behavior? | `git bisect` (below) |
| What did this merge actually change? | `git log --remerge-diff -1 <merge-sha>` (git >=2.36) |
| What changed between two versions of a rebased branch? | `git range-diff main old-tip new-tip` |
| Where did these two branches split? | `git merge-base --all main feature` |
| Which of my commits are not upstream yet? | `git cherry -v upstream` (matches by patch-id, survives rebases) |
| Who owns this area? | `git shortlog -sn --no-merges -- path/` |
| Anything else | `git log --oneline --graph --all -- path/` first; the shape of the history usually names the next command |

## Bisect

- Cost formula: steps = ⌈log2(commits in range)⌉ — 1,000 commits ≈ 10 tests, so bisect beats reading diffs on anything but a trivial range.
- Manual loop: `git bisect start` → `git bisect bad` (current) → `git bisect good v1.0.0` → test, answer `good`/`bad`, repeat → `git bisect reset`.
- Automated, the only version worth using when a test exists: `git bisect start HEAD v1.0.0 && git bisect run ./test.sh`. Exit 0 = good, 1–127 = bad, 125 = skip this commit (won't build).
- The test script must be robust to old commits: run it from a fixed path, reinstall dependencies inside it if the lockfile changes across the range, and `exit 125` on build failure rather than reporting `bad`.
- Flaky test → bisect converges on nonsense. Make the script run the test three times and only report `bad` on a consistent failure.
- Merge-heavy history: `git bisect start --first-parent` (git >=2.29) tests only mainline commits, so a bad commit is attributed to the merge that introduced it — much faster on a repo where every PR is a merge.
- Interrupted session: `git bisect log > bisect.log` saves it, `git bisect replay bisect.log` restores it. Answering wrong once is recoverable this way.
- Bisect works on any binary question, not just tests: performance thresholds, file size, "does this config still parse".

## Blame Beyond The Default

- Default blame reports the LAST touch, which after a reformat or a file move is meaningless. `-w` ignores whitespace-only changes, `-C -C -C` follows lines copied or moved from other files in the same commit and beyond.
- `git blame --ignore-rev <sha>` skips one commit; `git config blame.ignoreRevsFile .git-blame-ignore-revs` (git >=2.23) makes a committed list permanent, and GitHub honors the same file.
- `-L 40,80` or `-L :funcName:file` restricts to the region you care about; blaming a 3,000-line file whole wastes the run.
- Blame dead-ends at the commit that "added the file" after a rename Git failed to detect. `git log --follow -- path` crosses renames; `git log -S "the exact line"` finds the true origin when even that fails.

## Log Search Precision

- `-S` is a counting pickaxe: it reports commits where the NUMBER of occurrences changed. Moving a line between files makes counts cancel out and the commit disappears from the results — that is when `-G` (diff matches regex) is the right tool.
- `--all` searches every ref, including branches you never checked out; without it you only search the current branch's ancestry.
- `git log --oneline main..feature` = what the feature adds. `git log --left-right --oneline main...feature` = both divergences, marked with `<` and `>`.
- `--first-parent main` collapses to one line per merge — the PR-level history, with branch noise hidden.
- `--ancestry-path <sha>..HEAD` shows only commits that actually descend from that commit, which answers "did this fix reach the release branch".
- `git log --grep=<regex> --author=<name> --since=2.weeks` narrows before it searches; combining filters is cheap.

## Objects When Nothing Else Works

- `git rev-list --objects --all -- path` lists every object version of a path across all history, including branches deleted long ago.
- `git cat-file -p <sha>` prints any object: commit, tree, blob, tag. Reading a tree shows what the directory looked like without checking anything out.
- `git log --all --oneline --source -- path` prints which ref each match came from — the fastest way to find which abandoned branch still has the code you need.
- `git notes` attaches metadata (review links, CI results) to a commit without rewriting it; notes live in `refs/notes/*` and do NOT transfer on a normal clone — fetch them explicitly.
