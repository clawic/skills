# Compose Traps

`docker-compose` v1 is EOL (2023) — use the `docker compose` plugin; behavior below assumes v2. The development-loop side of Compose — watch mode, hot reload, seeded databases, debuggers — is in `development.md`.

**Before editing a stack that already exists**, read `## Stacks` and `## Volumes` in `~/Clawic/data/docker/memory.md` and any `artifacts/` compose file its `## Boxes` index names. The `start_period` that was tuned to a real boot time, and which volumes hold data that has never been restore-tested, are both there.

## Project Identity

- Project name defaults to the directory name: two checkouts in same-named folders share (and clobber) each other's containers, networks, and volumes. Set `name:` at the top of the file or `COMPOSE_PROJECT_NAME`.
- `docker compose up` does NOT rebuild when the Dockerfile or context changed — it reuses the existing image. `up -d --build` is the safe default during development.
- Renamed or removed services leave their old containers running: `down --remove-orphans`.

## depends_on

- `depends_on: [db]` waits for the container process, not for the service to accept connections — the canonical fix is `condition: service_healthy` (→ SKILL.md rule 8).
- `condition: service_healthy` without a healthcheck defined on the dependency fails at up-time — the two must ship together.
- `depends_on` does not apply to `docker compose run` — dependencies don't start.

## Environment

- Precedence for the container's env (high → low): `environment:` → `env_file:` → image ENV. For `${VAR}` interpolation in the YAML itself, your shell beats `.env`.
- `.env` is read only from the project directory (next to the compose file) — a subdirectory copy is ignored with no warning.
- Undefined `${VAR}` interpolates to empty string, not an error. `${VAR:?err}` makes it fail loudly; `${VAR:-default}` covers both unset AND empty, `${VAR-default}` only unset.
- `env_file` format is `KEY=value` lines only — `export KEY=value` breaks parsing.

## Volumes

- Bind mount over a populated image path: container files vanish; empty host dir = empty app dir. Named volumes seed from the image — on FIRST use only, never refreshed after.
- `./path` is relative to the compose file location, not your cwd.
- `down -v` deletes the project's named volumes — destructive; plain `down` keeps them.

## Networks

- Compose puts services on a project network with DNS by service name (container name differs — don't use it).
- `network_mode: host` disables port publishing and service DNS for that container entirely.
- `external: true` networks must already exist; compose never creates them.

## Healthcheck

- Defaults: `interval: 30s`, `timeout: 30s`, `retries: 3`, `start_period: 0s` — a service that needs 60s to boot is marked unhealthy before it ever answers; set `start_period` above worst-case boot time.
- During `start_period`, failed probes don't count against retries, but a passing probe immediately marks healthy.
- A compose healthcheck fully replaces the Dockerfile's; `disable: true` removes an inherited one.
- `test: ["CMD", ...]` needs the binary in the image (curl is absent in slim images — prefer `wget -qO-` or the app's own client); `CMD-SHELL` gets a shell. Exit 0 = healthy, 1 = unhealthy.

## Build

- The entire context uploads to the daemon before building — combined with a missing `.dockerignore` this is the usual "slow build" (→ `images.md` Size).
- `image:` + `build:` together = build locally and tag with that name; without `image:` the tag is `<project>-<service>`.
- YAML anchors don't cross files — for shared config use multiple `-f` files or `extends:`.

**When a compose file finally works**, save it to `~/Clawic/data/docker/artifacts/compose-<stack>.md` with a line saying when to read it and why the non-obvious parts are there — the `start_period` above worst-case boot, the anonymous volume shadowing a dependency directory, the `name:` that stops two checkouts clobbering each other. Add its `## Boxes` line to `memory.md` in the same turn, one row per service in `## Stacks`, and one row per named volume in `## Volumes` (`memory-template.md`). Every value that authenticates becomes a pointer before it is written: `POSTGRES_PASSWORD: <env:POSTGRES_PASSWORD>`.
