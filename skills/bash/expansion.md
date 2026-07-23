# Parameter Expansion Traps

- `${var:-default}` fires if unset OR empty — `${var-default}` only if unset. The set-but-empty case is where scripts diverge
- `${var:=default}` also assigns; `${var:-default}` doesn't touch var
- `${var:?msg}` aborts with msg if unset/empty — inline guard, the mechanism behind `rm -rf "${dir:?}/"` (→ SKILL.md Traps)
- `${var:-$(cmd)}` runs cmd whenever the default is needed — side effects included
- `${var: -3}` last 3 chars — the space is mandatory; `${var:-3}` is default-value syntax
- `${var:0:5}` counts characters in the current locale — under `LC_ALL=C` multibyte characters split into bytes

## Path surgery (worked example, `f=/a/b/c.tar.gz`)

- `${f##*/}` → `c.tar.gz` (basename) · `${f%/*}` → `/a/b` (dirname)
- `${f##*.}` → `gz` (extension) · `${f%%.*}` → `/a/b/c` — but if a DIRECTORY contains a dot (`/v1.2/c.tar.gz`) it eats the path too; strip the dir first: `b=${f##*/}; echo "${b%%.*}"` → `c`
- `#`/`%` = shortest from start/end; `##`/`%%` = longest. Patterns are globs, not regex: `*` wildcards, `.` is literal

## Replace and transform

- `${var/pat/rep}` first match — `${var//pat/rep}` all; empty replacement deletes: `${var//pat}`
- Strip Windows line endings after reading CRLF files: `${line//$'\r'/}`
- Case: `${var^^}` upper, `${var,,}` lower (bash >=4.0) — on macOS 3.2 pipe through `tr '[:upper:]' '[:lower:]'`
- Expansions don't nest: `${${f##*/}%.*}` is a syntax error — two assignments, two lines
- Per-element on arrays: `"${arr[@]/#/prefix-}"` prepends to every element; `"${arr[@]%.txt}"` strips from each

## Indirection

- `${!var}` expands the variable NAMED BY var — read-only indirection
- `declare -n ref=name` (bash >=4.3) — read AND write through the alias; prefer it over `eval` for dynamic names
- `${!prefix@}` lists variable NAMES starting with prefix, not values
- `${var@Q}` quotes a value safely for reuse in generated commands (bash >=4.4) — `printf '%q'` on older bash
