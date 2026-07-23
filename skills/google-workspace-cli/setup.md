# Setup — Google Workspace CLI

Read this on first use to load user preferences. Do not interview the user — answer the immediate request first; configuration accrues as the user reveals it.

If you have data at an old location (`~/google-workspace-cli/` or `~/clawic/google-workspace-cli/`), move it to `~/Clawic/data/google-workspace-cli/` before initializing anything.

## How To Load Preferences

1. Read `~/Clawic/data/google-workspace-cli/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults from the Configuration table in `SKILL.md` — do not ask:
   - `default_account`: the `gws auth default` account · `write_policy: dry-run-first` · `output_format: json` · `sanitize_mode: warn` · `mcp_services: drive,gmail,calendar`
3. Read `memory.md` for prior context (tenants, scope profiles, known-good templates). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about integration, boundaries, or how proactive to be.

## Initialize the Workspace (silent, on first write need)

```bash
mkdir -p ~/Clawic/data/google-workspace-cli
touch ~/Clawic/data/google-workspace-cli/{config.yaml,memory.md,command-log.md,change-control.md,incidents.md,mcp-profiles.md}
chmod 700 ~/Clawic/data/google-workspace-cli
chmod 600 ~/Clawic/data/google-workspace-cli/*
```

If `memory.md` is empty, seed it from `memory-template.md`. The tight permissions matter: these files name accounts, tenants, and command patterns.

## Recording Preferences (only when the user declares one)

- User names an account, output format, write posture, sanitize stance, or MCP bundle → update the matching key in `config.yaml`.
- User expresses a stance inside a preference area (tenant walls, scope policy, safety posture, conventions, automation cadence, no-go zones) → record it under that area in `config.yaml`; long texts get their own file in the folder, referenced by path.
- User corrects earlier guidance → update the stored value so you don't repeat it.
- Observed patterns (recurring failure signatures, working templates) go to `memory.md` — observations never overwrite a declared preference without user confirmation.

If the user has said nothing, store nothing.

## Operating Defaults Until Told Otherwise

- Inspect → dry-run → apply, one account and one tenant per operation batch (SKILL.md Rules 2 and 4).
- Minimal scopes; scope expansion is a recorded decision, not a reflex (`auth-playbook.md`).
- Command templates keep stable placeholders for ids so they are reusable from `command-log.md`.
- Every mutation runs through `change-control.md` gates.
