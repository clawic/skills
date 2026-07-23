# Setup - Google Workspace CLI

Read this when `~/clawic/google-workspace-cli/` is missing or empty.
Keep onboarding lightweight and task-first: answer the immediate request first, then improve future execution.

If you have data at the old `~/google-workspace-cli/` location, move it to `~/clawic/google-workspace-cli/` before initializing anything.

## First Activation Flow

1. Confirm integration behavior early:
- activate automatically for `gws` and Google Workspace API tasks, or explicit-only
- proactive suggestions allowed or not
- hard no-go scenarios (e.g. never send mail, never touch admin APIs)

2. Confirm operational context:
- individual productivity workflows
- team operations and reporting
- admin and compliance workloads (these need admin-privileged accounts — flag early, do not discover mid-task)

3. Confirm risk boundaries:
- dry-run required for all writes, or only for high-impact writes (send/share/delete)
- confirmation token required for send/share/delete operations
- production restrictions vs test tenant availability

4. Initialize local workspace if approved:
```bash
mkdir -p ~/clawic/google-workspace-cli
touch ~/clawic/google-workspace-cli/{memory.md,command-log.md,change-control.md,incidents.md,mcp-profiles.md}
chmod 700 ~/clawic/google-workspace-cli
chmod 600 ~/clawic/google-workspace-cli/{memory.md,command-log.md,change-control.md,incidents.md,mcp-profiles.md}
```

5. If `memory.md` is empty, initialize it from `memory-template.md`.

## Integration Defaults

- Default to inspect mode, then dry-run, then apply — one account and one tenant scope per operation batch.
- Prefer minimal scopes and explicit `--account` routing; scope expansion is a recorded decision, not a reflex.
- Keep command templates reusable with placeholders for ids.
- Track each mutating command with rationale and rollback notes in `change-control.md`.

## What to Save

- activation preferences and never-run boundaries
- default account and fallback account policy
- approved scope profiles by workflow type
- known-good command templates and failure signatures
- unresolved risks and mitigation decisions

## Guardrails

- Never ask users to paste credentials in chat.
- Never imply write safety without id resolution and preview.
- Never bypass tenant policy or OAuth verification requirements.
