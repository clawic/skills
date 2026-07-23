# Command Index — Services, Aliases, and Discovery

`gws` is dynamic: it builds resource/method commands from Google Discovery documents at runtime. The source of truth is always live introspection (`--help` + `schema`), never a memorized list — a static list would be stale by design.

## Service Alias Inventory

| Service alias | Alternate alias | API + version |
|---------------|-----------------|---------------|
| `drive` | - | `drive:v3` |
| `sheets` | - | `sheets:v4` |
| `gmail` | - | `gmail:v1` |
| `calendar` | - | `calendar:v3` |
| `admin` | `directory` | `admin:directory_v1` |
| `admin-reports` | `reports` | `admin:reports_v1` |
| `docs` | - | `docs:v1` |
| `slides` | - | `slides:v1` |
| `tasks` | - | `tasks:v1` |
| `people` | - | `people:v1` |
| `chat` | - | `chat:v1` |
| `vault` | - | `vault:v1` |
| `groupssettings` | - | `groupssettings:v1` |
| `reseller` | - | `reseller:v1` |
| `licensing` | - | `licensing:v1` |
| `apps-script` | `script` | `script:v1` |
| `classroom` | - | `classroom:v1` |
| `cloudidentity` | - | `cloudidentity:v1` |
| `alertcenter` | - | `alertcenter:v1beta1` |
| `forms` | - | `forms:v1` |
| `keep` | - | `keep:v1` |
| `meet` | - | `meet:v2` |
| `events` | - | `workspaceevents:v1` |
| `modelarmor` | - | `modelarmor:v1` |
| `workflow` | `wf` | synthetic workflow service |

Use `<api>:<version>` syntax to override a version explicitly (e.g., a beta surface).

## The Discovery Loop

1. `gws --help` — top-level syntax and global flags
2. `gws <service> --help` — resource and method branches for one service
3. `gws schema <service.resource.method>` — required params, body shape, per-API caps
4. `gws <service> <resource> [sub-resource] <method> --params '{...}' --json '{...}'` — the final command

Expected method not found: check the alias table above, remember discovery responses cache for 24 hours (a brand-new API method may not appear until the cache refreshes), and try the explicit `<api>:<version>` form.

## CLI Internals Worth Knowing

- Two-phase parsing: `gws` parses service + global flags first, then fetches discovery and rebuilds the command tree — which is why `--help` output differs per installation and why offline use fails on an empty cache.
- Global output formats: `json`, `table`, `yaml`, `csv` (set the user's `output_format` from config); pagination controls: `--page-all`, `--page-limit`, `--page-delay` (bounds in SKILL.md Rule 5).
- Credentials are encrypted (AES-256-GCM, keyring with file fallback) in `~/.config/gws/`; discovery cache lives in `~/.config/gws/cache/` with a 24-hour TTL.
- Structured JSON errors carry actionable fields — `enable_url` on `accessNotConfigured` is the fast path (→ SKILL.md Error Triage).
- `--sanitize <template>` integrates Model Armor screening for fetched content (SKILL.md Rule 6).
- Canonical repository: `googleworkspace/cli` — docs referencing `cliy` are the wrong, stale repo. The upstream generated index `docs/skills.md` lists helper/workflow shortcuts; locally, `gws workflow --help`.

## Exhaustive Service Sweep

Enumerate what exists right now in this installation:

```bash
for s in drive sheets gmail calendar admin admin-reports docs slides tasks people chat vault groupssettings reseller licensing apps-script classroom cloudidentity alertcenter forms keep meet events modelarmor workflow; do
  echo "===== $s ====="
  gws "$s" --help | sed -n '1,120p'
  echo
 done
```

## Safety Reminder

Discovery first, dry-run second, apply last. No write command without resolved stable ids and the gates in `change-control.md`.
