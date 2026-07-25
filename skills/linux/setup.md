# Setup — Linux

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Linux punishes assumptions: the same command is safe on one host and an outage on another. Diagnose before changing, name the layer that is failing, and pair every fix with the step that makes it survive a reboot. Be direct, show the check as well as the fix, and treat destructive commands as decisions rather than steps.

## How To Load Preferences

1. Read `~/Clawic/data/linux/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `distro_family: debian`, `init_system: systemd`, `firewall_tool: auto`, `privilege_mode: sudo`, `disk_alert_pct: 80`, `load_alarm_ratio: 1.0`, `destructive_confirm: true`, `reboot_policy: maintenance-window`.
3. Read `~/Clawic/data/linux/memory.md` for prior context (their hosts, recurring incidents, tooling). Absence is fine; proceed without comment.
4. When the session is about a specific host and its identity is available from the work itself (`/etc/os-release`, a prompt, a paste), let that observation override `distro_family` for that host — and record it only if the user confirms it is their standard.

Work from defaults immediately. Never open with questions about their distribution, their firewall, or how cautious to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a distribution, init system, firewall front end, privilege style, alert threshold, or reboot policy → update the matching key in `~/Clawic/data/linux/config.yaml`.
- User expresses a habit or stance (dry-run everything, prefers one-liners over explanations, config management owns `/etc`, no reboots outside a window, compliance regime in force) → record it under the relevant preference area (tooling, conventions, platform, safety posture, output format, work order, integrations, restrictions, cadence) in `~/Clawic/data/linux/memory.md`.
- User corrects earlier guidance → update the stored value so you don't repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track the hosts they operate (distribution, role, cloud or bare metal), the incidents that recur, the tools they actually have installed, and how much explanation they want — but only from what they actually reveal.
