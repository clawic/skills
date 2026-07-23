# Scripts — Auditing Executable Companions

Scripts are the most common payload location: scanners parse the markdown, humans skim the code. Every executable file gets read top to bottom — a 10-line script takes one minute, and that minute is the audit's best-spent one.

## Find Everything Executable

- By extension: `find <folder> -type f \( -name "*.sh" -o -name "*.bash" -o -name "*.py" -o -name "*.js" -o -name "*.mjs" -o -name "*.rb" -o -name "*.pl" \)`
- Extensionless with a shebang: `grep -rl '^#!' <folder>`
- Executable bit on a "data" file: `find <folder> -type f -perm +111` (BSD) / `-perm /111` (GNU)
- Compiled binaries, `.pyc`, `.so`, single-line minified JS = unauditable as shipped: reject unless the user explicitly accepts a checksummed, source-available artifact.

## Shell Red Flags

| Pattern | Why |
|---|---|
| `eval`, or a command assembled in a variable then run (`$cmd`, `bash -c "$x"`) | Executes strings the reader never saw assembled |
| `${var:offset:len}` slicing, `${!indirect}`, IFS games | Legit scripts don't build commands from character math |
| `printf '\x..'` / `echo -e` producing commands | Hex-assembled execution |
| Redefining common commands (`alias`, `function curl()`) | Every later line lies about what it does |
| `trap '...' EXIT` running network or write commands | Payload fires even when the visible flow looks clean |
| `2>/dev/null` wrapped around one specific block | Deliberate suppression marks exactly the lines to read hardest |
| Background `&` with `disown`/`nohup` | Outlives the session that was reviewed |

## Python Red Flags

`exec`/`eval`/`compile` on anything not a literal · `__import__`/importlib with computed names · `pickle.loads` on shipped data (arbitrary code on load) · `ctypes` in a "text helper" · `subprocess` with `shell=True` on built strings · `os.environ` harvesting beyond declared vars · requests/urllib to hosts missing from the egress ledger (`exfiltration.md`).

## Node Red Flags

`child_process` exec/spawn on built strings · `eval`/`Function()`/`vm` module · fetch/axios to undeclared hosts · `fs` writes outside the skill's own data folder · wholesale `process.env` reads · a shipped `package.json` with postinstall scripts.

## Constants Get Decoded

Voodoo constants — long numbers, packed strings, byte arrays — are content, not data to skip: decode per `hidden-content.md`. An "IV", "salt", or "lookup table" that decodes to a URL or a shell fragment ends the audit at reject.

## Writes and Self-Modification

- Legitimate write targets: the skill's own `~/Clawic/data/<slug>/` folder and files the user explicitly passes. Everything else — dotfiles, other skills' folders, agent config (settings, hooks, memory files), shell rc files, crontabs — is a persistence grab (`injection-patterns.md`): reject.
- A script that edits files in its own skill folder is self-modifying: the audit of version N cannot vouch for what version N wrote. Reject.

## Procedure

1. Inventory (above); executables join the read list.
2. Read each file fully — scripts have no skimming exemption.
3. Decode every constant that could hide a string.
4. Build the egress + write ledger per script → feeds Pass 3 findings.
5. Run nothing — not `--help`, not in a sandbox: audit is read-only (Rule 7), and a sandbox proves what the code did there, not what it does on the user's machine.
