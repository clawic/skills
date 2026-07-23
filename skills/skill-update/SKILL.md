---
name: skill-update
slug: skill-update
version: 1.0.1
description: Updates installed skills safely with diff preview, data migration, backup, and rollback. Use when updating, upgrading, or version-checking installed skills.
homepage: https://clawic.com/skills/skill-update
changelog: Clearer update workflow and safety checks
metadata:
  clawdbot:
    emoji: 🔄
    displayName: Skill Update
    configPaths:
    - ~/clawic/skill-update/
---

Updates installed skills safely: preview every change, back up before applying, migrate data only on approval, roll back on failure. Mode: act-as — it executes the update directly, never advises from the sideline. Backups and update state live under `~/clawic/skill-update/`.

## When To Use

- User asks to update, upgrade, or version-check an installed skill.
- A skill misbehaves right after an update and needs a rollback.
- A major bump is pending: preview breaking changes and migrate saved data before applying.
- User mentions a skill that has a newer published version — flag it once, proactively.
- Not for: publishing or versioning a skill you own (owner-side act → `skill-publish`); discovering or installing skills you don't have yet (→ `skill-finder`).

How updates break things (each maps to a preview check):
- Renamed/moved files → user data and references orphaned.
- Removed or rewritten instruction sections → the agent silently stops doing something the user relies on.
- New setup requirements (binary, env var, account) → skill half-works until configured.
- Changed data formats → previously saved state unreadable.

## Quick Reference

References (read on demand):
- `preview.md` — diff procedure, impact classification, recommend/defer call.
- `migrate.md` — data migration patterns, script template, failure handling.

Check installed against published:
```bash
npx clawic list              # installed skills + versions
npx clawic show <slug>       # latest published version
```
Catalog page mirrors the same version info: `https://clawic.com/skills/<slug>`.

Version delta sets the default stance (semver semantics):

| Delta | Assume | Action |
|---|---|---|
| Patch (1.2.3 → 1.2.4) | Fixes only | Quick diff, apply on approval |
| Minor (1.2 → 1.3) | Additions | Diff; check modified sections, not just added ones |
| Major (1.x → 2.0) | Breaking | Full preview + migration check; never propose-and-apply in the same turn |
| Any delta, user mid-project on this skill | Risky timing | Mention once, defer until the work ships |
| Else (delta unclear, or no version history) | Unknown risk | Treat as major |

## Core Rules

1. A skill is behavior-as-text: every text diff is a behavior diff. Never apply one the user hasn't seen and approved.
2. The update order is load-bearing — check → preview → explain → confirm → backup → update → verify (→ Update Flow). Back up before applying; verify before deleting anything.
3. Explicit "yes" only. Silence, "hmm", or a topic change is not consent. Approval for skill A is never approval for `update --all`.
4. Diff even patch bumps. Semver is the publisher's claim, not a guarantee: a "patch" that rewrites an instruction section is a behavior change wearing a small number.
5. Verify with one real task run against the updated skill, not a file listing. Files present ≠ skill works.
6. Back up before ANY update. Retention: keep ≥7 days AND until the user confirms the new version works — whichever is later. Breakage surfaces on next real use, not at install.
7. A migration that fails mid-way stops, reports, and restores — never leave partial state.
8. Any breaking-change signal leads the summary and requires explicit approval (signals → Preview Before Update).

## Update Flow

Order is load-bearing: backup before apply, verify before deleting anything.

1. **Check** — compare installed vs published version.
2. **Preview** — full diff, classified by impact (`preview.md`).
3. **Explain** — breaking changes first, in the user's workflow terms.
4. **Confirm** — explicit yes (Core Rule 3).
5. **Backup** — copy current state before touching anything.
6. **Update** — apply the new version.
7. **Verify** — run one real task with the updated skill.

## Preview Before Update

Never apply without showing impact. For each changed file answer: what changed, does the user's workflow touch it, is migration needed. Procedure and report format: `preview.md`.

Breaking-change signals — any one present means it leads the summary and requires explicit approval:
- File or folder renamed, moved, or deleted.
- Instruction section removed or rewritten (breaking even with zero structural change).
- New required setup step.
- Format change in anything the skill saves.

Present changes in a digestible summary, breaking items first:

> "Skill X has v2.0.0 available. Changes:
>
> **⚠️ Breaking:** Config now in `config.md` (was in SKILL.md)
> **Added:** New `templates/` folder with examples
> **Removed:** Old `legacy.md` no longer needed
>
> **Migration needed:** Your saved preferences need to move. I can help migrate. Proceed?"

Each skill with a pending major gets its own preview.

## Applying the Update

```bash
npx clawic update <slug>   # update one skill
npx clawic update --all    # update every installed skill with a newer version
```

`update` is the consumer-side pull — it fetches what the owner already published; publishing a new version is the owner's act, a different operation entirely (→ `skill-publish`). Reach for `--all` only when the user asked for all AND no pending update is a major (majors go one by one).

**Proactive notification:** when the user mentions a skill that has a newer version, say so once. Don't re-raise it in the same session.

## Backup Strategy

Before ANY update:
1. Copy the current skill folder to `~/clawic/skill-update/backups/<slug>-<version>-<timestamp>/`.
2. If the skill being updated declares a data folder (`~/clawic/<slug>/`), back that up too — the update itself never touches it, but a migration might.
3. State the backup path in your response.
4. If the update fails → offer restore immediately.

## Handling Migrations

When a data format changes:
1. Read the user's actual current data first — never map from an assumed format.
2. Explain what moves and what defaults get added.
3. Execute only with approval.
4. Verify migrated data works.

Patterns, script template, and failure handling: `migrate.md`.

## Rollback

Restore = copy the backup folder back over the installed skill. Offer it unprompted the moment the user reports the skill misbehaving after an update:

```
"Something's not working? I have a backup from before the update.
Want me to restore skill X to v1.2.3?"
```

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Trusting a patch bump | For text skills any edit changes behavior | Diff every update regardless of number |
| Verifying by file listing | Files present ≠ skill works | Run one real task with the updated skill |
| `update --all` after one yes | Consent covered one skill | Per-skill preview for every pending major |
| Deleting backup right after update | Breakage shows on next use, not at install | Hold ≥7 days and until user confirms |
| Flagging moved text as removed | Diff shows delete + add; content just relocated | Search the new version for "deleted" text before calling it breaking |

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/skill-update (install if the user confirms):
- `skill-finder` - Find, compare, and install skills you don't have yet, before there is anything to update.
- `skill-publish` - The owner-side act `update` pulls from: sanitize, version, and publish a skill you own.
- `skill-manager` - Broader install/remove/list lifecycle and unused-skill cleanup beyond one update.

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/skill-update.
