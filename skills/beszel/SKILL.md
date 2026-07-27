---
name: Beszel
slug: beszel
version: 1.0.0
description: Deploy, secure, and troubleshoot Beszel monitoring with Docker agents, alert tuning, and upgrade-safe operations for self-hosted servers
homepage: https://clawic.com/skills/beszel
changelog: Initial release with Beszel deployment guidance, alert tuning workflow, and incident troubleshooting playbooks.
metadata:
  clawdbot:
    emoji: 📊
    requires:
      bins:
      - docker
    os:
    - linux
    - darwin
    - win32
    configPaths:
    - ~/Clawic/data/beszel/
    displayName: Beszel
  openclaw:
    requires:
      config:
      - ~/Clawic/data/beszel/
---

## Setup

On first use, explain planned local storage in `~/Clawic/data/beszel/` and ask for confirmation before creating or updating files.

## When to Use

User needs help planning, deploying, or maintaining Beszel for infrastructure monitoring.
Agent handles topology choices, secure agent onboarding, alert calibration, and upgrade-safe operations.

## Architecture

Memory lives in `~/Clawic/data/beszel/`. See `memory-template.md` for setup.

```text
~/Clawic/data/beszel/
├── memory.md                  # Current environment, goals, and priorities
├── nodes.md                   # Agent inventory and ownership
├── alerts.md                  # Alert thresholds and escalation targets
└── incidents.md               # Incident timeline and recovery notes
```

## Quick Reference

| Topic | File |
|-------|------|
| Setup behavior | `setup.md` |
| Memory structure | `memory-template.md` |
| Deployment topologies | `deployment-patterns.md` |
| Alert calibration workflow | `alert-tuning.md` |
| Diagnostics and recovery | `diagnostics-playbook.md` |

## Data Storage

All skill files are stored in `~/Clawic/data/beszel/`.
Before writing new files, summarize the planned changes and get user confirmation.

## Core Rules

### 1. Define Monitoring Scope Before Installation
- Confirm which hosts, services, and failure modes matter most before recommending a layout.
- Prioritize high-impact systems first so coverage is meaningful from day one.

### 2. Keep Agent Access Minimal
- Use least privilege for monitored hosts and avoid broad root-level automation by default.
- Treat onboarding credentials as sensitive and avoid copying secrets into workspace notes.

### 3. Baseline First, Then Tune Alerts
- Collect a short baseline period before applying strict thresholds.
- Tune alert levels with operating context such as business hours, maintenance windows, and expected burst patterns.

### 4. Troubleshoot with a Fixed Sequence
- Validate service status, agent connectivity, clock sync, and resource pressure in a consistent order.
- Isolate one variable at a time to avoid changing many factors during an active incident.

### 5. Make Upgrades Reversible
- Require backup and rollback steps before version changes.
- Upgrade one environment at a time and confirm health before broader rollout.

### 6. Keep Operations Memory Current
- Record node ownership, alert intent, and previous incidents after meaningful changes.
- Convert repeated incidents into prevention rules and adjust thresholds accordingly.

## Common Traps

- Enrolling many agents before defining alert ownership -> noisy incidents with unclear responders.
- Copying thresholds from another environment -> chronic false positives or missed real failures.
- Upgrading hub and agents simultaneously -> hard-to-debug version mismatch issues.
- Ignoring host clock drift -> misleading timelines and confusing incident analysis.
- Logging sensitive credentials in notes -> unnecessary security exposure.

## Security & Privacy

**Data that may leave your machine (only when user configures external integrations):**
- Monitoring notifications sent to selected channels or alerting tools.
- Metadata required by external notification endpoints chosen by the user.

**Data that stays local by default:**
- Monitoring topology, node notes, threshold history, and incident logs in `~/Clawic/data/beszel/`.

**This skill does NOT:**
- Enable external alert destinations automatically.
- Create new credentials without explicit user approval.
- Send undeclared network requests.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `monitoring` — monitoring architecture and operational standards
- `server` — server diagnostics and maintenance workflows
- `self-host` — self-hosted deployment and hardening practices
- `docker` — container runtime and image management discipline
- `docker-compose` — multi-service orchestration patterns

## Feedback

- If useful, star it: https://clawic.com/skills/beszel
- Latest version: https://clawic.com/skills/beszel
