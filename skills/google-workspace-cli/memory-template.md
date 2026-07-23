# Memory Template — Google Workspace CLI

Files live in `~/Clawic/data/google-workspace-cli/`. Split: `config.yaml` holds what the user DECLARED (the Configuration table in SKILL.md); `memory.md` holds what the agent OBSERVED. An observation never overwrites a declared preference without user confirmation.

## config.yaml

```yaml
default_account:            # email; falls back to `gws auth default`
write_policy: dry-run-first # dry-run-first | confirm-only | open
output_format: json         # json | table | yaml | csv
sanitize_mode: warn         # warn | block | off
mcp_services: [drive, gmail, calendar]
# preference areas — keys added as the user states preferences:
# accounts_and_tenants: {}
# scope_policy: {}
# safety_posture: {}
# conventions: {}
# automation_cadence: {}
# no_go_zones: []
```

## memory.md

```markdown
# Google Workspace CLI Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Environment Context
- accounts seen and their tenants
- tenant_type: personal | team | enterprise
- admin privileges available: yes | no | unknown

## Scope Profiles
- profile name -> scopes and allowed services
- recorded scope expansions and why

## Working Templates
- known-good command templates (stable id placeholders)
- recurring failure signatures and their fixes

## Open Risks
- unresolved auth issues
- API enablement gaps
- policy or compliance blockers

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning | Behavior |
|-------|---------|----------|
| `ongoing` | Context still evolving | Keep refining boundaries and templates |
| `complete` | Stable operating baseline | Focus on optimization and reliability |
| `paused` | User paused this workflow | Keep context read-only until resumed |
| `never_ask` | User does not want setup prompts | Do not ask integration questions unless requested |

## Companion Files

- `command-log.md` — command template, required placeholders, expected output fields, known side effects, run counts (`automation.md`)
- `change-control.md` — the mutation evidence log; entry template in this skill's `change-control.md`
- `incidents.md` — failures, root causes, prevention actions
- `mcp-profiles.md` — service bundles per workflow family and tool budget decisions (`mcp-integration.md`)
