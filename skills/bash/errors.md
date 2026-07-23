# Error Handling Traps

The three `set -e` killers practitioners actually hit — `local` masking, `((i++))`, `grep -q` SIGPIPE — live in SKILL.md Traps. Mechanics below.

## Where set -e is blind

- Not in an `if`/`while` condition — and that immunity extends to the ENTIRE body of any function called there: `if myfunc; then` runs all of `myfunc` with `-e` off
- Not after `||` or `&&` — `cmd || true` is the idiomatic "allowed to fail"
- Not inside `$(cmd)` in an assignment — the subshell's failure vanishes; fix with `shopt -s inherit_errexit` (bash >=4.4)
- `exit` in a subshell exits only the subshell — the parent continues

## Pipelines

- Exit code is the LAST command's — `bad | good` returns 0 without `set -o pipefail`
- `pipefail` says a pipeline failed but not which segment — read `PIPESTATUS`, and copy it immediately: `st=("${PIPESTATUS[@]}")` — any command, even `[[`, resets it
- Early-exit consumers (`head`, `grep -q`) close the pipe; the producer dies with SIGPIPE = exit 141 (128+13) — under pipefail that reads as failure even when the pipeline did its job
- Variables set inside a pipe segment don't persist — subshell (→ SKILL.md Subshells and State)

## Traps and exits

- `trap cleanup EXIT` fires on error and normal exit alike; `kill -9` bypasses every trap — make cleanup idempotent (`rm -f`, not `rm`)
- A second `trap ... EXIT` REPLACES the first silently — one handler per script, or chain manually inside it
- `ERR` traps don't fire inside functions or subshells unless you `set -E` (errtrace)
- `return` outside a function is an error — `exit` at top level, `return` in functions
- Redirect order: `>file 2>&1` sends both to file; `2>&1 >file` sends stderr to the OLD stdout (the terminal)
- `$( )` captures stdout only — stderr passes through to the terminal; capture both with `$(cmd 2>&1)` only when you truly want them merged

## Reporting

- `die() { printf '%s\n' "$*" >&2; exit 1; }` — errors go to stderr so `$(...)` callers don't capture them as data
- Include context in the message: `die "config not found: $cfg"` — the failing VALUE, not just the fact
