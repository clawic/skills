---
name: skill-manager
slug: skill-manager
version: 1.0.3
description: 'Manage installed skills lifecycle: suggest by context, track installations, check updates, and cleanup unused.'
homepage: https://clawic.com/skills/skill-manager
changelog: 'Fix contradictions: clarify declined tracking, add npx security note'
metadata:
  clawdbot:
    emoji: 🧩
    displayName: Skill Manager
---

## Skill Lifecycle Management

Manage the full lifecycle of installed skills: discovery, installation, updates, and cleanup.

**References:**
- `suggestions.md` — when to suggest skills based on current task
- `lifecycle.md` — installation, updates, and cleanup

**Complements:**
- `skill-finder` — user-initiated search ("find me a skill for X")
- `skill-manager` — proactive lifecycle management

---

## Scope

This skill ONLY:
- Suggests skills based on current task context
- Tracks installed skills in `~/Clawic/data/skill-manager/inventory.md`
- Tracks skills user explicitly declined (with their stated reason)
- Checks for skill updates

This skill NEVER:
- Counts task repetition or user behavior patterns
- Installs without explicit user consent
- Reads files outside `~/Clawic/data/skill-manager/`

---

## Security Note

This skill uses `npx clawic` commands which download and execute code from the Clawic catalog. This is the standard mechanism for skill management. Always review skills before installing.

---

## Context-Based Suggestions

When working on a task, notice the **current context**:
- User mentions specific tool (Stripe, AWS, GitHub) → check if skill exists
- Task involves unfamiliar domain → suggest searching

This is responding to current context, not tracking patterns.

## Lifecycle Actions

| Action | Command |
|--------|---------|
| Install | `npx clawic add <slug>` |
| Update one | `npx clawic update <slug>` |
| Update everything | `npx clawic update --all` |
| Info | `npx clawic show <slug>` |
| Remove | Delete the skill's folder from each detected agent directory (`.claude/skills/`, `.codex/skills/`, etc.) — `npx clawic` has no uninstall command yet |

Each skill also has a canonical catalog page at `https://clawic.com/skills/<slug>` for a quick look before installing or updating.

---

## Memory Storage

Inventory at `~/Clawic/data/skill-manager/inventory.md`.

**First use:** `mkdir -p ~/skill-manager`

**Format:**
```markdown
## Installed
- slug@version — purpose — YYYY-MM-DD

## Declined
- slug — "user's stated reason"
```

**What is tracked:**
- Skills user installed (with purpose and date)
- Skills user explicitly declined (with their stated reason)

**Why track declined:** To avoid re-suggesting skills user already said no to. Only stores what user explicitly stated.
