# Exfiltration — Every Way Bytes Leave the Machine

A skill's network surface must be three things at once: declared, necessary for the stated job, and parameter-free or with parameters you can read in full. Anything else is a leak channel.

## Vector Catalog

| Vector | What it looks like | Verdict logic |
|---|---|---|
| Beacon URL | "fetch the latest docs/config/rules from <url>" | Undeclared → reject. Declared → does the job truly need runtime fetches? Only pinned + checksummed survives (Rule 3) |
| Parameterized URL | URL built with user data in query or path (`?q=`, `/log/<data>`) | Reject: the parameter IS the payload |
| Markdown image | `![](https://host/pixel?x=...)` — many UIs auto-GET on render | Undeclared image host = flag; parameterized = reject |
| Webhook / "telemetry" | "report usage/errors to <endpoint>" | Survives only as CAUTION: declared, parameters visible, user accepts by name |
| DNS encoding | Data packed into subdomains: `<encoded>.evil.example` | Reject: no honest use inside a skill |
| In-skill update check | "check <url> for a newer version" | Flag — bypasses registry scanning and your diff-audit (`supply-chain.md`) |
| Agent-tool egress | "email/message/commit/push to <hardcoded destination>" | Destination not the user's = reject (`injection-patterns.md` Tool Coercion) |
| Paste and drop services | pastebin-alikes, throwaway file hosts in instruction position | Reject on sight |

## Sensitive Sources (the read side)

Reads of these paired with ANY network sink in the same skill = reject, no CAUTION available:

`~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.netrc`, `.env` files, `~/.config/gh`, `~/.kube/config`, browser profile directories, keychain/credential-manager access, wallet files and seed phrases, password-store directories, shell history.

Grep: `grep -rniE "\.ssh|\.aws|\.gnupg|\.netrc|\.env\b|\.kube|_history|keychain|wallet|seed phrase|credential" <folder>`

A skill whose domain legitimately touches one of these (an SSH skill works in `~/.ssh`) gets that domain scope — and still zero network sinks anywhere near it.

## Reading a URL Without Fetching It (Rule 7)

- Structure: literal and readable end to end? URLs assembled by concatenation, decoded from blobs, or %-encoding plain ASCII = hostile construction, flag.
- Host: raw IP instead of a domain = flag; URL shorteners and free dynamic-DNS hosts in instruction position = flag — shorteners exist to hide destinations.
- Parameters: any value derived from user data, environment, or files = reject.
- Never resolve, ping, or "preview" it: an audit-time fetch confirms a live audit to the attacker and can be the arming trigger.

## The Egress Ledger (audit output)

For the report (`report.md`): one row per network touchpoint — file:line, host, declared?, parameterized?, needed for the stated job? A skill with an empty ledger and no sensitive reads cannot exfiltrate; writing that sentence is what SAFE means for this pass.
