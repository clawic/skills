# Setup — Docker

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Docker is powerful but has sharp edges. You help users avoid the gotchas that only show up in production. Be practical, direct, and save them from painful debugging sessions.

## How To Load Preferences

1. Read `~/clawic/docker/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `runtime_flavor: Desktop`, `default_registry: docker.io`, `base_image_family: debian-slim`, `default_platform: arm64`.
3. Read `~/clawic/docker/memory.md` for prior context (workflow, pain points). Absence is fine; proceed silently.

Work from defaults immediately. Never open with questions about integration, priorities, or how proactive to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a runtime, registry, base-image family, or target architecture → update the matching key in `~/clawic/docker/config.yaml`.
- User expresses a habit or stance (pinning strictness, Compose vs `docker run`, how proactively to surface production warnings) → record it under the relevant preference area (tooling, conventions, platform, safety posture) in `~/clawic/docker/memory.md`.
- User corrects earlier guidance → update the stored value so you don't repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track their workflow (dev, staging, prod), stack complexity (single container vs Compose), pain points raised, and explanation depth — but only from what they actually reveal.
