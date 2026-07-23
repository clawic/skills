# Sweep — Auditing the Installed Fleet

Point-of-install audits decay: skills update, new agents get added, folders accumulate. A sweep re-establishes that what is on disk is what was approved.

## Where Skills Live

Each agent loads skills from its own directories; the sweep covers all that exist on the machine (restrict with `sweep_scope`):

- Project-level: `.claude/`, `.codex/`, `.gemini/`, `.openclaw/`, `.agents/`, `.cursor/` skill folders inside repos the user works in
- Global: the same agents' home-directory equivalents (e.g. `~/.claude/skills/`) plus `~/.hermes/`
- Discovery: `find ~ -maxdepth 4 -type d -name skills 2>/dev/null | grep -E "\.(claude|codex|gemini|openclaw|agents|cursor|hermes)"` — then repeat inside active project roots

## Triage Order (never full-audit everything)

1. Mechanical Pass 1 + Pass 2 greps (`checks.md`) across ALL folders in one run — grep scales, reading doesn't.
2. Full five-pass audit only on: any grep hit; any skill absent from the audit log; any skill whose on-disk content differs from the version last audited.
3. Priority among full audits: skills containing scripts > skills with network endpoints > text-only — execution outranks prose in risk order.

## Drift Detection

- Same slug in several agent directories at different versions: audit the newest; older copies either update or inherit the newest verdict with a note.
- On-disk edits with no registry release behind them: something wrote into a skill folder → `incident.md`.
- Orphans — folders matching no registry listing and no log entry: treat as pasted-folder installs, full audit.

## Sweep Output

- One line per skill: slug, version, location(s), verdict — or "unchanged since <date> audit" when the log covers it.
- Log the sweep date in `audit-log.md` (`report.md` format).
- The cadence preference area decides when a sweep is due; when the skill activates and the log shows it overdue, say so in one line. Mention, never nag.
