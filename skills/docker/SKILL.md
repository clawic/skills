---
name: Docker
slug: docker
version: 1.0.9
description: >-
  Builds, debugs, hardens, and ships Docker containers, images, and Compose stacks. Use when writing or reviewing
  a Dockerfile, a compose file, or a CI build step; when a container exits instantly, restart-loops, is
  OOM-killed, hangs on stop, or exits 137/139/127; when a published port is unreachable, containers cannot resolve
  each other, or requests hang behind a VPN; when the disk fills and `/var/lib/docker` will not prune; when a
  build is slow, the cache never hits, or fails only in CI; when `exec format error` or a musl-versus-glibc break
  is the problem; when choosing a base image or a multi-stage layout; when a registry login or pull rate limit
  fails; when a secret must stay out of image history; and when volumes need backup, restore or a permission fix.
  Covers Compose traps and Desktop/colima/OrbStack/Podman differences. Not for Kubernetes manifests or cluster
  scheduling (`k8s`).
homepage: https://clawic.com/skills/docker
changelog: "Clearer data layout: what to read, what to save, and where"
metadata:
  clawdbot:
    emoji: 🐳
    requires:
      bins:
      - docker
    os:
    - linux
    - darwin
    - win32
    displayName: Docker
    configPaths:
    - ~/Clawic/data/docker/
    - ~/Clawic/data/servers/
---

**Data.** At the start of every session, read `~/Clawic/data/docker/config.yaml` (what the user declared) and `~/Clawic/data/docker/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Read `~/Clawic/data/servers/servers.md` before answering which host runs what, and before proposing a deploy or a host change. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever it produced something durable: a Docker host discovered, rebuilt or retired; a stack, image, or base-image decision; a volume and its backup or restore result; a deploy and the digest that would roll it back; an environment fact that cost effort to find (VM memory ceiling, VPN MTU, corporate CA, registry mirror, port already taken); a failure whose cause was not obvious; or something the user will re-read — a Dockerfile or compose file that finally worked, a `daemon.json`, a runbook. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Docker hosts go to the shared inventory `~/Clawic/data/servers/servers.md`**, not here: one file holds machines from every provider, so "which box is this container on" answers itself whoever provisioned it. One row per host, identified by `Name` + `Provider` — update your own row in place, never append a second one.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in text the user pastes in to be saved. Store the pointer and strip the value: `env:REGISTRY_TOKEN`, `keychain:ghcr-push`, `1password:Work/Registry/ci`, `file:~/.docker/config.json`. If data sits at an old location (`~/docker/` or `~/clawic/docker/`), move it to `~/Clawic/data/docker/`.

Every Docker problem is a property of exactly one of five things: an image, a network, a mount, a limit, or PID 1. Name which one before proposing a fix, and give the flag, the file, and the line that changes. Work from defaults immediately: never open with questions about their runtime, their registry, or how proactive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals) → the Configuration table default.

## When To Use

- Writing or reviewing Dockerfiles, Compose files, or container build steps in CI
- Debugging containers: crashes, restart loops, OOM kills, unreachable ports, DNS failures, slow or non-reproducible builds
- Reclaiming disk, capping logs, backing up volumes, or hardening containers and daemons for production
- Choosing base images, pinning strategy, multi-stage layout, or a per-language Dockerfile recipe
- Registry work: login and credential helpers, rate limits and mirrors, digest promotion, retention, signing
- Setting up or fixing the local development loop: hot reload, seeded databases, attached debuggers, devcontainers
- Not for Kubernetes manifests or cluster scheduling (`k8s`) — this covers image building plus single-host and Compose-level operation, including for images that later run on Kubernetes

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| Container exits instantly | `docker logs <id>` (works on dead containers), then `docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}'` | `debug.md` |
| Exit code 137 | `OOMKilled=true` → raise `-m` or fix the leak; `false` → external SIGKILL, usually stop-timeout expiry (Rule 4) | `debug.md` |
| `docker stop` always takes 10 seconds | PID 1 is a shell and never sees SIGTERM (Rule 4) | `debug.md` |
| Host can't reach container | App must bind `0.0.0.0` inside the container AND the port must be published | `networking.md` |
| Container can't reach host | `host.docker.internal`; Linux Engine needs the host-gateway flag | `networking.md` |
| Containers can't resolve each other | They are on the default bridge — DNS only works on user-defined networks | `networking.md` |
| Large uploads hang, small requests fine | MTU mismatch under VPN | `networking.md` |
| Build slow or cache always misses | Layer order (deps before code) + `.dockerignore`, then cache mounts | `images.md` |
| Disk filling up | `docker system df -v` to locate the leak, then targeted prune (→ Disk Leaks) | `production.md` |
| Code change not appearing | `docker compose up -d --build` — plain `up` reuses the stale image | `compose.md` |
| `exec format error` | CPU architecture mismatch (arm64 image on amd64 host or vice versa) — build with `--platform` | `ci.md` |
| Works locally, fails in CI | Architecture, image digest, env vars, bind mounts, filesystem case — in that order | `debug.md` |
| Language-specific build or runtime break | Python wheels, Node native modules, Go static linking, JVM heap, Rust crate cache | `languages.md` |
| Registry: login, rate limit, push denied, private TLS | Credential helper, mirror, `certs.d`, digest promotion | `registry.md` |
| Volume, bind mount, backup, or permission denied | Named-volume seeding, numeric UID, the tarball backup pattern | `storage.md` |
| Hot reload, seeded DB, or debugger not attaching | Watch mode, `initdb.d` only-when-empty, source-path mapping | `development.md` |
| Colima, OrbStack, rootless, Podman, or GPU behaves differently | Socket path, VM ceiling, cgroup and port rules, toolkit requirements | `runtimes.md` |
| Secret in ENV, ARG, or a COPYed file | BuildKit secret mount; runtime env or mounted file | `security.md` |
| Taking it to production, or rolling back | daemon.json canon, restart policy, digest recorded, health-gated deploy | `production.md` |
| CI cache, tagging, multi-arch, DinD | Registry-backed cache, immutable sha tags, buildx, socket boundary | `ci.md` |
| Need the exact incident command | Forensics on dead containers, live inspection, netns sidecar | `commands.md` |
| Anything else Docker | Reproduce with a minimal `docker run` and zero flags, re-add flags one at a time; the flag that breaks it names the subsystem | — |

Coverage map: `debug.md` symptom→cause · `commands.md` incident toolkit · `images.md` build and cache · `languages.md` per-runtime Dockerfiles · `compose.md` Compose traps · `development.md` the local loop · `networking.md` reachability, DNS, firewall, MTU · `storage.md` volumes, mounts, backup · `registry.md` registries, auth, retention, signing · `runtimes.md` Desktop/colima/OrbStack/rootless/Podman/GPU · `production.md` daemon, deploys, monitoring · `security.md` hardening and secrets · `ci.md` cache, tagging, multi-arch.

## Core Rules

1. **Pin what you can't afford to re-debug.** Dev: minor tag (`python:3.11-slim`). Prod and CI: digest (`python@sha256:...`) — tags are mutable, digests are not. Governed by `pin_policy`. Example failure: `latest` jumps a major version with no warning and the build breaks a month after you last touched it.
2. **Order layers by change frequency.** Dependency manifest → install → source code. `COPY . .` before the install step invalidates the dependency cache on every code edit — the single largest build-time waste in real projects.
3. **`apt-get update && apt-get install -y pkg` in one RUN.** Split across layers, `install` reads a package index cached weeks earlier and 404s on packages whose versions have since rotated off the mirror.
4. **Exec-form CMD; respect PID 1.** Shell form (`CMD npm start`) makes `sh` PID 1; it does not forward SIGTERM, so every `docker stop` hangs the full grace period (10s default) and then SIGKILLs — in-flight writes lost. Use `CMD ["npm","start"]` or `--init`. Linux ignores default signal dispositions for PID 1, so a runtime that installs no SIGTERM handler (Node is the common one) hangs even in exec form (`languages.md`).
5. **Non-root with a numeric UID.** `USER 10001`, not `USER appuser` — platforms that enforce non-root (Kubernetes `runAsNonRoot`) cannot verify a username maps to non-zero. Place `USER` after the RUNs that need root.
6. **Memory limits: know the swap formula.** `-m 512m` alone allows swap = 2× memory, so the real ceiling is 1 GiB. Hard cap: set `--memory-swap` equal to `--memory` (`-m 512m --memory-swap 512m` = no swap). Then cap the runtime inside it at ~75% of that (`languages.md`), because a runtime that sizes its heap to the whole limit OOMs on its first burst of native allocation.
7. **Cap logs at run time.** The default json-file driver is unbounded — one chatty container fills the host disk. `--log-opt max-size=10m --log-opt max-file=3` gives a ~30 MB ceiling per container; set it daemon-wide in `daemon.json` so nobody forgets.
8. **Gate startup on health, not on start.** Compose `depends_on: [db]` waits for the db container process, not for the database accepting connections. Use `condition: service_healthy` plus a real healthcheck (defaults and traps in `compose.md`).
9. **Record the digest at deploy, or you have no rollback.** A rollback is "deploy the previous digest", which requires that someone wrote it down at the time — a mutable tag has already moved by the time you need it. Capture `docker inspect -f '{{index .RepoDigests 0}}' <image>` at deploy and write the row to `deploys/<year>.md` (`memory-template.md`) in the same turn. Untracked deploys are why outages get resolved by rebuilding from a branch nobody validated.

## Exit Codes

Formula: a code above 128 means killed by signal `code − 128`.

| Code | Meaning | First move |
|------|---------|-----------|
| 125 | Docker daemon error (bad flag, missing image) | Read the run command, not the app |
| 126 | File found but not executable | `chmod +x`; or the entrypoint is a directory; or CRLF line endings on the entrypoint script |
| 127 | Command not found | PATH wrong, or a glibc binary on musl (Alpine), or the shell itself is absent (distroless) |
| 137 | SIGKILL (128+9) | Check `.State.OOMKilled`; also fired by stop-timeout expiry |
| 139 | SIGSEGV (128+11) | Native crash — suspect glibc/musl or architecture mismatch |
| 143 | SIGTERM (128+15) | Clean external stop — usually not a bug |

## Defaults That Decide Behavior

Docker's defaults are tuned for a laptop demo, not for a service. Each of these has produced a production incident that reads as an application bug.

| Default | Value | Why it bites |
|---|---|---|
| Log driver | `json-file`, unbounded | One chatty container fills the host disk and hangs the daemon (Rule 7) |
| `/dev/shm` | 64 MB | Chrome/Playwright and Postgres parallel queries crash with obscure errors; `--shm-size=1g` is the fix, not more RAM |
| Stop grace period | 10s, then SIGKILL | Anything that needs longer to drain must set `--stop-timeout`/`stop_grace_period` AND actually handle SIGTERM |
| Restart backoff | 100 ms, doubling, capped at 1 min | A crash loop self-throttles; rising `RestartCount` is the alarm, not the log volume |
| Default bridge network | No embedded DNS | Container-name resolution fails; user-defined networks have it (`networking.md`) |
| Healthcheck | interval 30s, timeout 30s, retries 3, `start_period` 0s | A service that boots in 60s is marked unhealthy before it ever answers (`compose.md`) |
| Address pool | 172.17.0.0/16 onward | Collides with corporate VPN ranges; set `default-address-pools` (`production.md`) |
| Network MTU | 1500 | Under a VPN with a lower MTU, large payloads hang while small ones succeed (`networking.md`) |
| User | root | Unless the image or `USER` says otherwise (Rule 5) |
| pids limit | unlimited | One fork bomb takes the host, not the container; `pids_limit: 256` is cheap insurance |

## Disk Leaks

`/var/lib/docker` at 100% hangs the daemon itself — you cannot prune through a daemon that won't respond. Alert well before full; locate leaks with `docker system df -v`.

| Leak | Reclaim |
|------|---------|
| Dangling images | `docker image prune` |
| Build cache | `docker builder prune --keep-storage <build_cache_budget_gb>GB` |
| Stopped containers | `docker container prune`, or `--rm` at run time |
| Named volumes | `docker volume prune` — NOT touched by `system prune` without `--volumes`; destructive, confirm first |
| Orphan networks | `docker network prune` |
| A container's own writable layer | Not prunable — the app is writing inside the container instead of a volume; find it with `docker diff <c>` (`storage.md`) |

## Output Gates

Before emitting a Dockerfile, a Compose file, or a deploy command:

- Base image pinned to the strictness `pin_policy` requires — tag at minimum, digest if this ships to prod?
- Dependency install layered before source copy, and a `.dockerignore` present excluding `.git` and dependency directories?
- CMD/ENTRYPOINT in exec form, and does this runtime actually handle SIGTERM as PID 1?
- `USER` set with a numeric UID, or root explicitly justified?
- No secret reachable via ENV, ARG, or a COPYed file — and none written into `~/Clawic/data/`?
- Memory limit with a matching `--memory-swap`, a log cap, and the runtime's own heap capped below the container limit?
- Compose: healthcheck defined on every service something depends on, `start_period` above worst-case boot time?
- Is the command destructive (`prune`, `down -v`, `volume rm`, `system prune --volumes`)? Then it names exactly what dies and ships with a confirmation step, never inside a copy-paste block of read-only commands.
- Did anything durable come out of this — a host, a deploy digest, a working file, a volume, an environment fact, a root cause? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/docker/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| runtime_flavor | Desktop \| colima \| orbstack \| rootless \| podman | Desktop | Selects socket path, `host.docker.internal` behavior, VM-ceiling reasoning and default-platform assumptions; rootless and podman change port-binding, cgroup and volume-permission advice (`runtimes.md`) |
| default_registry | text (registry host) | docker.io | Prefixes unqualified image references; switches push/pull and mirror examples to the user's registry (`registry.md`) |
| base_image_family | alpine \| debian-slim \| distroless | debian-slim | Drives base-image recommendations and the Alpine-vs-slim tradeoff (musl wheels, DNS, no-shell debugging) in `images.md` and `languages.md` |
| default_platform | arm64 \| amd64 | arm64 | Sets the assumed build/run architecture; governs `--platform` reminders, multi-arch advice in `ci.md`, and `exec format error` diagnosis |
| pin_policy | tag \| digest-in-prod \| digest-everywhere | digest-in-prod | The strictness Rule 1 and the Output Gates enforce on every generated Dockerfile and compose file |
| hardening_profile | default \| hardened | default | `hardened` makes every generated run/compose ship `--read-only`, `--cap-drop ALL`, `no-new-privileges` and a non-root UID unprompted (`security.md`) |
| ci_platform | github-actions \| gitlab-ci \| jenkins \| buildkite \| none | none | Which CI dialect `ci.md` examples are written in, and which cache backend is recommended |
| build_cache_budget_gb | number (GB, 1-200) | 10 | The `--keep-storage` figure in every prune command and the threshold for calling build cache a disk problem |
| destructive_confirm | bool | true | Whether `prune`, `down -v` and `volume rm` are emitted behind an explicit confirmation step or inline |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — Compose vs plain `docker run`, buildx/bake vs classic build, devcontainers vs a hand-rolled dev compose file, testcontainers usage — affects which shape every example takes
- **Conventions** — tag scheme (`sha`, semver, date), image namespace and labels, `.dockerignore` habits, one-file-vs-many compose layout — affects generated files and `ci.md` tagging
- **Platform** — single host vs CI vs a cloud target, multi-arch need, registry mirror or pull-through cache, host OS family (SELinux hosts need `:z`/`:Z`) — affects `storage.md` and `registry.md`
- **Safety posture** — how proactively to surface production hardening, whether to emit destructive commands at all, deletion confirmations, appetite for `--privileged` escapes — affects Output Gates and `security.md`
- **Cadence** — prune schedule, base-image rebuild and rescan frequency, volume-restore drill, reboot drill — every accepted cadence becomes a row in the `## Due` table of `memory.md`
- **Output register** — command-first vs explanation-first, whether to show the diff of a Dockerfile or the whole file, how much of the reasoning to keep — affects every answer's shape

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Secrets via ENV, ARG, or COPY | All three persist in image history — deleting the file in a later layer does not remove the earlier layer | BuildKit `RUN --mount=type=secret`; runtime env or mounted files (`security.md`) |
| Mounting `/var/run/docker.sock` | Socket access = root on the host; any container escape is total | Dedicated proxy with a filtered API, or rethink the design |
| `ADD` for local files | Auto-extracts archives; URL downloads bypass the build cache | `COPY`; fetch URLs in a RUN with checksum verification |
| `docker logs` shows nothing | Only PID 1's stdout/stderr is captured — and buffered runtimes hold it until they exit | Log to stdout unbuffered (`PYTHONUNBUFFERED=1` and friends, `languages.md`), or symlink the logfile to `/dev/stdout` |
| No shell in distroless/slim image | Nothing to `exec` into | `docker cp` files out, or attach a sidecar sharing the netns (`commands.md`) |
| `--privileged` to fix a permission error | Disables every isolation mechanism at once | Find the one capability or device needed (`security.md`) |
| Bind mount over an image path | Host dir replaces container contents; empty host dir = empty app dir | Named volume — it seeds from the image on first use (`storage.md`) |
| `restart: always` on dev boxes | Containers you stopped by hand come back after every host reboot | `unless-stopped` |
| Chasing an app "memory leak" without checking the VM | On Desktop/colima the VM has its own ceiling; the container never saw the RAM you think you gave it | `docker info` Total Memory before touching `-m` (`runtimes.md`) |
| `chmod 777` on a bind mount | Makes the symptom go away and the data world-writable; the mismatch is numeric UID, not permission bits | `COPY --chown=<uid>:<gid>` at build and run as that UID (`storage.md`) |
| Rebuilding to roll back | The branch you rebuild from is not the artifact that was validated | Deploy the recorded digest (Rule 9) |
| Scanning only in CI | CVEs are published after the build; an image approved in March rots in place | Scan at build AND in the registry, and rebuild on a cadence (`security.md`, `## Due`) |
| A base-image or hardening decision that lives only in the chat | Re-litigated every quarter by whoever is on call | `artifacts/` with the date and what was rejected (`memory-template.md`) |

## Where Experts Disagree

- **Alpine vs debian-slim.** Alpine is smaller but musl breaks prebuilt Python wheels and has DNS edge cases. Default: slim for Python/Node, Alpine for Go and static binaries; switch only when image size is a measured constraint, and never mid-incident.
- **Compose in production.** Legitimate for single-host deployments; the boundary is multi-host scheduling, rolling deploys, or autoscaling — those needs, not fashion, justify an orchestrator (`k8s`).
- **One process per container.** The default. Escape hatch: a process supervisor when the platform offers no sidecar mechanism — never as a convenience to avoid writing a second service.
- **Distroless vs a debuggable base.** Distroless removes the shell an attacker would use and the shell you would use at 3am. The frontier is whether you can reliably attach a sidecar in the environment where it will break: if you can, distroless; if production is a box you SSH into, a slim base with a non-root user wins on mean-time-to-recovery.
- **Rootless as the default.** Rootless removes the largest single risk (daemon-as-root) at the cost of privileged ports, some mount types, and slower storage on older kernels. Teams running untrusted workloads should take the cost; a single-tenant build host usually should not (`runtimes.md`).

## Security & Privacy

**Credentials:** this skill drives the Docker CLI, which reads registry credentials from `~/.docker/config.json` or an OS credential helper. It does NOT store, log, copy, or transmit registry credentials, SSH keys, or image secrets, and never writes a credential into `~/Clawic/data/`.

**Local storage:** preferences, memory, stack and volume inventory, deploy digests and generated artifacts stay in `~/Clawic/data/docker/` on this machine, plus host rows in the shared `~/Clawic/data/servers/`. Image names, digests, ports and volume names only — no secrets.

**Guardrails:** commands are read-only by default. Destructive operations (`prune`, `down -v`, `volume rm`, `system prune --volumes`) name exactly what they delete and require explicit confirmation when `destructive_confirm` is true.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/docker (install if the user confirms):
- `k8s` — Kubernetes manifests and cluster debugging, once the images leave a single host
- `devops` — deployment pipelines and release process around the build
- `linux` — host system management, cgroups, systemd units, firewalls
- `server` — server administration for the box the daemon runs on
- `traefik` — reverse proxy and TLS in front of Compose services

## Feedback

- If useful, star it: https://clawic.com/skills/docker
- Latest version: https://clawic.com/skills/docker

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/docker.
