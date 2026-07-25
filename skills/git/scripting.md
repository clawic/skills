# Scripting Git — Plumbing, Exit Codes, and Unattended Runs

Porcelain commands are written for humans and their output is not a contract; plumbing commands and the explicit machine formats are. Anything running without a person watching — CI, hooks, an agent — uses the second set.

## Human Command → Script Command

| Question | Do not parse | Use instead |
|---|---|---|
| Which branch am I on? | `git branch` (the `*`, the detached-HEAD line) | `git branch --show-current` (git >=2.22) — empty output means detached |
| Is there anything to commit? | `git status` | `git diff --quiet && git diff --cached --quiet` |
| Full working-tree state | `git status` | `git status --porcelain=v2 --branch -z` |
| Does this ref exist? | grepping `git show` output | `git rev-parse --verify --quiet <ref>^{commit}` |
| Which files are tracked? | `ls`, `find` | `git ls-files -z` or `git ls-tree -r --name-only HEAD` |
| A file's content at some revision | check out, then read | `git show <rev>:<path>` — no checkout, index untouched |
| Every branch with its upstream state | a loop over `git branch -vv` | `git for-each-ref --format='%(refname:short) %(objectname) %(upstream:track)' refs/heads/` |
| Is A already contained in B? | `git log \| grep` | `git merge-base --is-ancestor A B` |
| Anything else | — | Look for a `--porcelain` or `--format` flag in `git help <cmd>` before writing a parser |

- `--porcelain` is a stability promise: the format stays put across Git versions, unlike the human output, which changes whenever the UX improves.
- `-z` NUL-terminates records. Without it Git quotes paths containing spaces, quotes, or newlines, and `core.quotePath` escapes non-ASCII into `\303\251` — two ways for a filename to break a loop that worked on the author's machine.
- `--porcelain=v2` adds what v1 leaves out: the branch header (`# branch.ab +2 -1`), rename sources, and file modes. Parse v2 whenever ahead/behind counts or renames matter.
- `git rev-parse` is the universal resolver: `--show-toplevel` (work tree root), `--git-dir`, `--git-common-dir` (shared state across worktrees), `--is-inside-work-tree`, `--abbrev-ref HEAD`. Deriving these with `pwd` and string surgery is where portable scripts go wrong.

## Exit Codes Worth Branching On

| Command | 0 means | 1 means | Other |
|---|---|---|---|
| `git diff --quiet` / `--exit-code` | no difference | difference found | >1 = the command itself failed |
| `git merge-base --is-ancestor A B` | A is an ancestor of B | it is not | 128 = a ref does not resolve |
| `git rev-parse --verify --quiet <ref>` | SHA on stdout | missing, no output | — |
| `git check-ref-format --branch <name>` | Git will accept the name | rejected | — |
| `git ls-remote --exit-code --heads origin <b>` | the branch exists server-side | — | 2 = no matching ref |
| `git grep -q <pattern>` | matched | no match | 128 = bad revision or path |

Git returns 128 for fatal errors and 129 for usage errors. A blanket `|| true` swallows a mistyped flag exactly as cheerfully as a legitimate "nothing to do" — test the specific code, or capture it into a variable and dispatch.

## Running Without A Terminal

```bash
GIT_TERMINAL_PROMPT=0                 # fail fast instead of blocking on a credential prompt
GIT_ASKPASS=/bin/true                 # same, for the GUI-style prompt some helpers raise
GIT_EDITOR=true                       # accept a generated message or rebase todo list as-is
GIT_SEQUENCE_EDITOR='sed -i.bak s/^pick/fixup/2'   # rewrite the rebase todo list programmatically
GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_SYSTEM=/dev/null   # git >=2.32: hermetic run, user config cannot change behavior
GIT_OPTIONAL_LOCKS=0                  # read-only commands stop writing the index — for pollers and shell prompts
GIT_AUTHOR_NAME  GIT_AUTHOR_EMAIL  GIT_AUTHOR_DATE        # identity and dates per invocation,
GIT_COMMITTER_NAME  GIT_COMMITTER_EMAIL  GIT_COMMITTER_DATE   # without writing any config file
```

- `git -c <key>=<value>` overrides one setting for one command; `git -C <dir>` runs it elsewhere without `cd`. Both propagate to the subprocesses Git spawns.
- Committing fails with "unable to auto-detect email address" when no identity is configured. In a runner, supply it through the variables above rather than writing a global config the next job inherits.
- Git only pipes to a pager on a TTY, but a half-attached CI shell can still hang on one: add `--no-pager` (or `GIT_PAGER=cat`) to any command whose output you capture.
- Most CI checkouts are detached, so `git branch --show-current` returns empty and `@{u}` errors out. Read the CI's own ref variable for the branch name and `git rev-parse HEAD` for identity; never infer the branch from the checkout.
- Hermetic runs matter for reproducibility: a developer's `pull.rebase`, `merge.conflictStyle`, or a mergetool set globally changes what the same script does on their laptop versus the runner.

## Concurrency

- The index lock is per worktree: two processes in one checkout collide on `.git/index.lock`, two worktrees never do. That is the mechanical argument for one worktree per parallel job.
- Ref updates take a per-ref lock and fail with "cannot lock ref" rather than corrupting anything. `git update-ref --stdin` applies a whole batch of ref changes as one transaction — all of them or none.
- A rejected push in automation is a lost race far more often than a real divergence: fetch, rebase, retry with a bounded count, fail the job after that. Never resolve a race with `--force`.
- Commands you think are read-only still write: `gc --auto` fires on commit and fetch, and `status` refreshes the index. On a shared mirror several jobs read, set `gc.auto=0` and `GIT_OPTIONAL_LOCKS=0`.

## Committing Without A Working Tree

```bash
blob=$(git hash-object -w file.txt)                          # write the content into the object store
tree=$(printf '100644 blob %s\tfile.txt\n' "$blob" | git mktree)
commit=$(git commit-tree "$tree" -p HEAD -m "generated")
git update-ref refs/heads/generated "$commit"                # publish atomically
```

No checkout, no index, no working tree to clean up, and no local hooks in the path. This is how a bot commits against a bare repo, and the reason a server-side automation never needs `git clone` at all.
