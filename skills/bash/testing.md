# Testing Traps

## [ vs [[

- `[ $var = "x" ]` breaks when var is empty or has spaces — quote inside `[ ]` always; `[[ $var = x ]]` doesn't word-split, unquoted left side is safe
- `<` and `>` inside `[ ]` are redirects — you just created a file named `9`. Use `-lt`/`-gt`, or `[[ ]]` where they compare
- `[[ $a < $b ]]` is still LEXICAL string comparison — `[[ 10 < 9 ]]` is true (1 sorts before 9). Numbers go in `(( ))` or `-lt`
- `==` and `=` are identical inside `[[ ]]`; POSIX `[` only guarantees `=`
- `[[ -v var ]]` (bash >=4.2) is true for set-but-EMPTY variables — `[[ -n ${var:-} ]]` is not; pick by which case you mean

## Patterns and regex

- Glob match only in `[[ ]]`: `[[ $f == *.txt ]]` — the pattern side must be UNQUOTED; `"*.txt"` matches a literal asterisk
- Regex: keep the pattern in a variable — `re='^[0-9]+ (GET|POST)'; [[ $line =~ $re ]]`. Quoting any part of an inline pattern makes that part literal, the classic silent regex breakage
- Captures land in `BASH_REMATCH`: `${BASH_REMATCH[1]}` = first group
- `case` patterns are globs with `|` alternation — `*.txt|*.md)` works, regex doesn't; `;&` falls through (bash >=4.0)

## Arithmetic

- `(( n > 5 ))` for numbers — but a variable holding a non-numeric string silently evaluates to 0, and untrusted content can execute code via subscripts (→ SKILL.md Core Rule 8)
- Leading zeros mean octal: `$((08))` is an error ("value too great for base") — force decimal with base prefix: `m=08; echo $((10#$m))` → 8
- `(( ))` returns status 1 when the value is 0 — the `((i++))` + `set -e` interaction (→ SKILL.md Traps)

## File tests

- `-e` exists (any type) · `-f` regular file · `-d` directory · `-L` symlink itself (`-f` follows the link)
- `-r`/`-w`/`-x` test YOUR effective permissions, not the mode bits
- `-s` exists AND non-empty — the common "did the command produce output" check
- `file1 -nt file2` newer-than — rebuild-if-stale logic without make
