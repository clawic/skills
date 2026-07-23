# Array Traps

- `arr=($(cmd))` splits on whitespace AND globs — `mapfile -t arr < <(cmd)` (bash >=4.0); on 3.2, `while IFS= read -r l; do arr+=("$l"); done < <(cmd)`
- `mapfile` without `-t` keeps a trailing `\n` on every element — always `-t`
- Associative arrays need `declare -A` (bash >=4.0) — without it, string keys silently evaluate to index 0 and overwrite each other
- Associative iteration order is unspecified — sort `"${!map[@]}"` when output must be stable
- `${#arr}` is the length of element 0 — `${#arr[@]}` is the count
- Unquoted `${arr[@]}` splits and globs — always `"${arr[@]}"`; keys: `"${!arr[@]}"`
- Empty array + `set -u`: `"${arr[@]}"` errors on bash <4.4 — portable guard: `${arr[@]+"${arr[@]}"}`
- `arr+=(item)` appends an element — `arr+=item` string-appends to element 0, no error
- `for i in {1..$n}` — brace expansion runs before variable expansion, you get the literal `{1..3}` — use `for ((i=1; i<=n; i++))`
- `unset arr[2]` — quote it: `unset 'arr[2]'`, or a file named `arr2` glob-matches; the index gap stays, indices never shift
- Sparse gaps: `copy=("${arr[@]}")` re-indexes 0..n and loses original indices — iterate `"${!arr[@]}"` if indices carry meaning
- `${arr[-1]}` needs bash >=4.3 — portable last element: `${arr[@]: -1}` (the space is required; without it `:-` becomes default-value syntax)
- Slices: `${arr[@]:1:2}` — elements 1 and 2; works on `$@` too: `"${@:2}"` = args from the 2nd on
- Pass an array into a function: `f "${arr[@]}"` then `local items=("$@")`; by reference: `declare -n ref=$1` (bash >=4.3) — the nameref must not share the caller's variable name or bash errors with "circular name reference"
- Expansions apply per element: `"${arr[@]%.txt}"` strips the suffix from every element
