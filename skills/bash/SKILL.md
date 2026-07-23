---
name: bash
slug: bash
version: 1.0.4
description: >-
  Writes reliable Bash scripts: quoting, arrays, error handling, traps, and
  version portability. Use when writing or debugging shell scripts, one-liners,
  cron jobs, or CI pipelines.
homepage: https://clawic.com/skills/bash
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🖥️
    requires:
      bins:
      - bash
    os:
    - linux
    - darwin
    displayName: Bash
---

## When To Use

- Writing or reviewing any Bash beyond a one-liner: CI jobs, deploy scripts, cron tasks, glue code
- Debugging scripts that break on spaces in filenames, empty variables, or fail silently
- Hardening an existing script: strict mode, cleanup on exit, portability across bash versions
- Deciding whether the task belongs in Bash at all (see Core Rule 5)
- Not for POSIX-sh-only targets (dash, busybox, alpine `/bin/sh`) — most patterns here are bashisms

## Quick Reference

| Situation | Go to |
|-----------|-------|
| Breaks on spaces or hostile filenames | Quoting below, then `testing.md` |
| Looping over files, lines, or command output | Robust Iteration below, `arrays.md` |
| `set -e` missed a failure, cleanup never ran | `errors.md` |
| String surgery: defaults, trim, replace, basename | `expansion.md` |
| Comparison wrong: `[` vs `[[`, numeric vs lexical, regex | `testing.md` |
| Must run on macOS stock bash or old servers | Version Floors below |
| Anything else | Core Rules below |

## Core Rules

1. Open every script with `#!/usr/bin/env bash` and `set -euo pipefail`, then learn the `-e` holes (`errors.md`) instead of dropping strict mode — the holes are enumerable; silent failures are not.
2. Quote every expansion: `"$var"`, `"$(cmd)"`, `"${arr[@]}"`. An unquoted expansion is a deliberate act that carries a comment saying why. Word splitting plus globbing is Bash's #1 bug class (shellcheck SC2086).
3. Run shellcheck before shipping. Suppress only with the code and a reason on the same line: `# shellcheck disable=SC2086 -- flags must split`.
4. Know your floor: macOS `/bin/bash` is 3.2 forever (GPLv3 freeze). If the script uses any `bash >=4.0` feature (Version Floors table), state the floor in a header comment.
5. Past ~100 lines or once you need nested data structures, rewrite in Python or similar — the Google Shell Style Guide's cutoff. Bash excels at orchestrating processes, not at logic.
6. Never parse `ls`. Iterate with globs or `find -print0`: filenames may contain newlines, so NUL is the only safe delimiter.
7. Test the failure path before delivering: swap one command for `false`, confirm the script stops, the trap fires, and the exit code is nonzero. A cleanup you never saw run is a cleanup you don't have.
8. Untrusted input never reaches arithmetic or `eval`: `(( $userinput ))` executes commands via `arr[$(cmd)]` subscripts. Gate with a regex first: `[[ $n =~ ^[0-9]+$ ]] || die "not a number: $n"`.

## Script Skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail
shopt -s inherit_errexit 2>/dev/null || true   # bash >=4.4: $(cmd) failures propagate

die() { printf '%s\n' "$*" >&2; exit 1; }

tmp=$(mktemp) || die "mktemp failed"
trap 'rm -f "$tmp"' EXIT   # fires on error and normal exit; kill -9 bypasses all traps
```

## Quoting

- `"$var"`, `"$(cmd)"`, `"${arr[@]}"` — always. `$(cmd)` strips ALL trailing newlines, not just one.
- Single quotes are literal; `$'...'` interprets escapes: `$'\t'`, `$'\r'`, `$'\0'`.
- Filenames from variables get `--` or `./`: `rm -- "$f"` survives a file named `-rf`.
- `echo "$var"` breaks when var is `-n`, `-e`, or has backslashes — `printf '%s\n' "$var"` never does.
- Arguments to `ssh`/`su -c` are re-parsed by the remote shell — build them with `printf '%q '`.
- `${arr[*]}` joins with first char of IFS into one word; `"${arr[@]}"` preserves elements. Joining is the only reason to write `[*]`.

## Version Floors

| Feature | Needs |
|---------|-------|
| `declare -A`, `mapfile`, `${var^^}`/`${var,,}`, `;&` fallthrough | bash >=4.0 |
| `[[ -v var ]]`, `shopt -s lastpipe` | bash >=4.2 |
| `${arr[-1]}`, `declare -n` namerefs | bash >=4.3 |
| `inherit_errexit`, `${var@Q}`, empty `"${arr[@]}"` safe under `set -u` | bash >=4.4 |

macOS `/bin/bash` stays at 3.2. `#!/usr/bin/env bash` finds a Homebrew bash on PATH; `#!/bin/bash` never will.

## Subshells and State

- Every pipe segment runs in a subshell: `cmd | while read -r x; do ((n++)); done` loses `n`. Fix: `done < <(cmd)`, or `shopt -s lastpipe` (bash >=4.2, scripts only).
- `( )` is a subshell, `{ ...; }` is the current shell — `exit` inside `( )` or `$( )` exits only that subshell.
- Background jobs: `cmd & pid=$!` then `wait "$pid"` — `wait` returns the job's exit code, your only way to check it.
- `cd` inside `( )` to visit a directory without having to `cd` back.

## Robust Iteration

- Globs: `shopt -s nullglob` first — otherwise `for f in *.txt` in an empty dir runs once with the literal string `*.txt`.
- Hostile filenames or recursion: `while IFS= read -r -d '' f; do ...; done < <(find . -name '*.log' -print0)`.
- Lines of a file: `while IFS= read -r line; do ...; done < file` — `IFS=` keeps leading whitespace, `-r` keeps backslashes. A final line without trailing newline is still skipped: append `|| [[ -n $line ]]` to the read.
- Split a string: `IFS=, read -ra fields <<< "$csv"`. Join: `(IFS=,; echo "${arr[*]}")` — the subshell keeps the IFS change local.

## Output Gates

Before delivering any script, check:

- Every expansion quoted, or the unquoted one carries a comment saying why
- shellcheck exits 0, or each disable names its SC code and reason
- Failure path exercised: injected `false`, watched the trap fire and the exit code go nonzero
- Bash floor stated in a header comment if any `bash >=4.0` feature is used
- No `eval`; no unvalidated input inside `(( ))` or array subscripts

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| `local out=$(cmd)` | `local` returns 0, masking cmd's failure — `set -e` never fires | `local out; out=$(cmd)` |
| `((count++))` when count is 0 | expression evaluates to 0 → exit status 1 → `set -e` kills the script | `count=$((count+1))` |
| `grep -q` downstream under pipefail | early exit sends SIGPIPE upstream; producer dies with 141 (128+13) and the pipeline "fails" on success | capture first: `out=$(cmd)`, then grep the variable |
| `rm -rf "$dir/"` | empty/unset `dir` → `rm -rf /` | `rm -rf "${dir:?}/"` aborts if empty |
| Checking `$?` after a log line | the `echo` overwrote it | `rc=$?` on the very next line |
| `which cmd` to test existence | external, output format varies (SC2230) | `command -v cmd >/dev/null` |
| `cd "$dir"` without a check | without `-e`, everything after runs in the wrong directory | `cd "$dir" || exit 1` — habit survives scripts that lack `-e` |

## Where Experts Disagree

- `set -e`: the strict-mode school makes it mandatory; the Google Shell Style Guide school argues its exceptions (conditions, `||`, command substitution) make it false comfort and prefers explicit `|| die`. Boundary: short glue scripts → strict mode; sourced libraries and functions whose return codes callers inspect → explicit handling. Never mix philosophies in one file.
- Bash vs POSIX sh: write sh only when the target set actually contains dash/busybox/alpine. "Portable by default" costs arrays, `[[ ]]`, and `set -o pipefail` for hosts you may never meet.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/bash (install if the user confirms):

- `linux` — when the bug is the system, not the script: permissions, cron, systemd, OOM
- `regex` — when the `=~` pattern itself is the hard part
- `github-actions` — when the script lives in CI and the failure is workflow wiring, not shell

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/bash.
