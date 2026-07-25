# Hooks — Local Automation That Actually Runs

A hook is an executable file Git runs at a lifecycle point. Non-zero exit aborts the operation for the "pre-" hooks; stdout and stderr reach the user.

## The Two Facts That Explain Every Hook Problem

1. **`.git/hooks` is never versioned and never installed by a clone.** That is a security property, not an oversight: cloning a repo must not execute its code. Sharing hooks therefore requires an explicit opt-in step.
2. **`--no-verify` bypasses them.** Hooks are a convenience for the author, never a control. Anything that must hold is enforced server-side (branch protection, required checks, push rules) — hooks just move the feedback earlier.

## Sharing Hooks

```bash
mkdir -p .githooks && git config core.hooksPath .githooks   # committed dir, one command per clone
```

Put the `git config` line in the project's setup script or `make bootstrap`; a README instruction that everyone must remember has a predictable adoption rate. Hook files need the executable bit committed (`git update-index --chmod=+x .githooks/pre-commit`), or they are ignored on clone with no error.

## Which Hook For Which Job

| Hook | Fires | Good for | Cost model |
|---|---|---|---|
| `pre-commit` | Before the message editor | Formatters, fast linters on staged files only | Runs on every commit — budget it in single-digit seconds |
| `prepare-commit-msg` | Before the editor opens | Injecting a ticket ID from the branch name, templates | Must be idempotent: amend re-runs it |
| `commit-msg` | After the message is written | Subject-length and convention checks | Cheap; the natural home for conventional-commit validation |
| `pre-push` | Before the network call | Test suite, type check, secret scan | The right place for anything slow — it runs once per push, not per commit |
| `post-checkout` / `post-merge` | After the tree changes | Reinstall dependencies when the lockfile changed | Cannot abort anything; keep it advisory |
| `pre-rebase` | Before a rebase starts | Refusing to rebase a protected branch | Rarely used, occasionally exactly right |

Worked example — ticket ID from the branch name, added only when missing:

```bash
#!/bin/sh
# .githooks/prepare-commit-msg
case "$2" in message|commit|merge|squash) exit 0 ;; esac   # only for a fresh, editor-written message
id=$(git symbolic-ref --short HEAD | grep -oE '[A-Z]+-[0-9]+') || exit 0
grep -q "$id" "$1" || printf '%s: %s' "$id" "$(cat "$1")" > "$1"
```

## Traps

- **A formatter that rewrites files in `pre-commit` must re-stage them.** The commit uses the index as it was; files fixed on disk but not `git add`-ed produce a commit with unformatted content and a working tree that immediately looks dirty. Either `git add` the touched paths inside the hook, or use a runner that handles partial staging.
- **Partially staged files are the hard case.** If a file is half-staged (`add -p`), a hook that formats the whole file mixes unstaged work into the commit. Frameworks solve this by stashing the unstaged remainder; hand-rolled hooks usually do not.
- **Two frameworks fight over `core.hooksPath`.** husky and the `pre-commit` framework both set it; installing both leaves whichever ran last in control and the other dead with no error. Check `git config core.hooksPath` when a hook "stopped working".
- **Hooks must not be interactive.** A prompt, a pager, or a password read blocks any non-TTY caller — CI, an IDE, an automated agent — with no output at all (`errors.md`, "Nothing at all happens").
- **Hooks inherit a minimal environment** from GUI clients: a hook that works in your terminal fails in the IDE because `node`/`python` are not on that PATH. Resolve interpreters explicitly or source the version manager inside the hook.
- **`$GIT_DIR` differs in a worktree.** Use `git rev-parse --show-toplevel` for the source tree and `--git-common-dir` for shared state; hardcoded `.git/` paths break for every colleague using worktrees (`worktrees.md`).
- **Server-side merges never run your hooks.** A squash-merge performed by the host produces a commit message no `commit-msg` hook ever saw — so convention enforcement has to exist in CI as well if it matters.
- **Slow hooks get bypassed, then removed.** Anything over a few seconds on `pre-commit` trains everyone to type `--no-verify`; move it to `pre-push` before that habit forms.
- **`git commit --amend` re-runs `pre-commit` and `commit-msg`** — a non-idempotent hook (appending a trailer every time) produces duplicated lines.

## Debugging A Hook

```bash
sh -x .githooks/pre-commit          # run it directly with tracing; hooks are just scripts
GIT_TRACE=1 git commit -m test      # confirms whether Git even invoked it
git config core.hooksPath           # empty means .git/hooks; wrong value means a framework took over
ls -l .git/hooks/                   # sample files end in .sample and never run
```

Most "my hook does not run" reports are one of three things: missing executable bit, `.sample` suffix still attached, or `core.hooksPath` pointing elsewhere.

## Scanning Gates Worth Installing

- Secret scanning on `pre-commit` catches the leak before it costs a rotation (`secrets.md`) — the highest-value hook most repos lack.
- Large-file guard on `pre-commit`: reject staged blobs above the host's limit — or above the smaller size recorded in the thresholds preference area — before the push fails server-side (`large-repos.md`).
- Branch guard on `pre-push`: refuse pushes to `main` from a machine where that should never happen — the cheapest protection on a repo without server-side rules.
