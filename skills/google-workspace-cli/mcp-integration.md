# MCP Integration — Exposing gws to Agents

`gws mcp` serves API methods as MCP tools over stdio. The design decision is always the same: which services, for which workflow, with what write exposure — never `-s all` by reflex.

## Start the Server

```bash
# Narrow exposure (default bundle from `mcp_services` in config)
gws mcp -s drive,gmail,calendar

# Include workflow shortcuts
gws mcp -s drive,gmail -w
```

Client configuration:

```json
{
  "mcpServers": {
    "gws": {
      "command": "gws",
      "args": ["mcp", "-s", "drive,gmail,calendar"]
    }
  }
}
```

## Tool Budget Discipline

- Each service expands to many tools; agents pick worse as tool count grows, and some clients cap the list outright. Budget from the task backward: which methods does THIS workflow call? Expose those services only.
- Profile per workflow family, not per session: a `mail-triage` profile (`-s gmail`), a `file-ops` profile (`-s drive`), a `reporting` profile (`-s drive,sheets,admin-reports`). Record each profile and its rationale in `mcp-profiles.md` so the next session reuses instead of re-deriving.
- If a client still struggles with the count, split one workflow across two profiles before reaching for fewer services than the task needs.

## Write Exposure

- Read-only investigation → expose only services whose write paths the agent must not have, or pair the session with `write_policy` gates: the MCP tools execute with the same account and scopes as the CLI, so scope choice (`auth-playbook.md` readonly tiers) is the real write barrier — a readonly-scoped token makes every exposed write tool fail safely.
- The change-control gates (`change-control.md`) apply to agent-invoked writes exactly as to human-typed ones; an agent with MCP access and `write_policy: open` is the one combination that needs an explicit user decision first.

## Content Safety in Agent Loops

An agent consuming Gmail bodies or Doc contents through MCP is the highest-risk shape for prompt injection (SKILL.md Rule 6): the content flows straight into an autonomous loop. Run with `--sanitize` per `sanitize_mode`; prefer `block` over `warn` when the agent acts without human review between read and write.

## Session Checklist

- Service list matches the current workflow profile (validate before starting)
- Account explicit and correct for the tenant — the MCP server inherits auth like any gws invocation (SKILL.md Rule 4)
- Write-capable scopes only when the workflow mutates
- Profile logged in `mcp-profiles.md`; workflow shortcuts (`-w`) only when actually used — some workflow MCP calls are still partial upstream
