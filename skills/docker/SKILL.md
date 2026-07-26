---
name: Docker
slug: docker
version: 1.0.7
description: Builds, debugs, and hardens Docker containers, images, and Compose stacks. Use when writing Dockerfiles or compose files, when a container crashes, restart-loops, gets OOM-killed, cannot reach the network, or fills the disk, when builds are slow or fail only in CI, when choosing base images or pinning, or when preparing containers and hosts for production.
homepage: https://clawic.com/skills/docker
changelog: "Complete coverage map: debugging, networking, storage, production, and CI guides"
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
---

User preferences and memory live in `~/Clawic/data/docker/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/docker/` or `~/clawic/docker/`), move it to `~/Clawic/data/docker/`.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/docker/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| runtime_flavor | Desktop \| colima \| orbstack \| rootless | Desktop | Selects socket path, `host.docker.internal` behavior, and default-platform assumptions; rootless changes port-binding and volume-permission advice |
| default_registry | string (registry host) | docker.io | Prefixes unqualified image references and pin/digest guidance; switches examples to the user's mirror or private registry |
| base_image_family | alpine \| debian-slim \| distroless | debian-slim | Drives base-image recommendations and the Alpine-vs-slim tradeoff (musl wheels, DNS, no-shell debugging) |
| default_platform | arm64 \| amd64 | arm64 | Sets the assumed build/run architecture; governs `--platform` reminders and `exec format error` diagnosis |

Preference areas to record as the user reveals them:

- **tooling** — Compose vs plain `docker run`, BuildKit/bake usage, preferred prune cadence
- **conventions** — pinning strictness (tag vs digest), layer/`.dockerignore` habits, naming and label standards
- **platform** — single-host vs CI vs prod target, multi-arch build needs, registry and mirror setup
- **safety posture** — how proactively to surface production hardening (non-root, log caps, secret handling) vs only on request

## When To Use

- Writing or reviewing Dockerfiles, Compose files, or container build steps in CI
- Debugging containers: crashes, restart loops, OOM kills, unreachable ports, DNS failures
- Reclaiming disk, capping logs, or hardening containers for production
- Choosing base images, pinning strategy, or multi-stage layout
- Not for Kubernetes manifests or cluster scheduling — this covers single-host and Compose-level operation

## Quick Reference

| Situation | Play |
|-----------|------|
| Container exits instantly | `docker logs <id>` (works on dead containers), then `docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}'` |
| Exit code 137 | `OOMKilled=true` → raise `-m` or fix the leak; `false` → external SIGKILL, usually stop-timeout expiry (rule 4) |
| Host can't reach container | App must bind `0.0.0.0` inside the container AND the port must be published |
| Container can't reach host | `host.docker.internal` (`networking.md` for the Linux flag) |
| Containers can't resolve each other | They are on the default bridge — DNS only works on user-defined networks |
| Build slow or cache always misses | Layer order (deps before code) + `.dockerignore` → `images.md` |
| Disk filling up | `docker system df -v` to locate the leak, then targeted prune (→ Disk Leaks) |
| Code change not appearing | `docker compose up -d --build` — plain `up` reuses the stale image |
| `exec format error` | CPU architecture mismatch (arm64 image on amd64 host or vice versa) — build with `--platform` |
| Anything else | Reproduce with a minimal `docker run`, then re-add flags one at a time until it breaks |

Depth on demand: `debug.md` symptom→cause playbooks · `commands.md` incident toolkit · `images.md` build and cache · `compose.md` Compose · `networking.md` reachability, DNS, firewall, MTU · `storage.md` volumes, mounts, backup · `production.md` daemon config, deploys, monitoring · `security.md` hardening and secrets · `ci.md` cache, tagging, multi-arch.

## Core Rules

1. **Pin what you can't afford to re-debug.** Dev: minor tag (`python:3.11-slim`). Prod and CI: digest (`python@sha256:...`) — tags are mutable, digests are not. Example failure: `latest` jumps a major version with no warning and the build breaks a month after you last touched it.
2. **Order layers by change frequency.** Dependency manifest → install → source code. `COPY . .` before the install step invalidates the dependency cache on every code edit — the single largest build-time waste in real projects.
3. **`apt-get update && apt-get install -y pkg` in one RUN.** Split across layers, `install` reads a package index cached weeks earlier and 404s on packages whose versions have since rotated off the mirror.
4. **Exec-form CMD; respect PID 1.** Shell form (`CMD npm start`) makes `sh` PID 1; it does not forward SIGTERM, so every `docker stop` hangs the full grace period (default 10s) and then SIGKILLs — in-flight writes lost. Use `CMD ["npm","start"]` or `--init`.
5. **Non-root with a numeric UID.** `USER 10001`, not `USER appuser` — platforms that enforce non-root (Kubernetes `runAsNonRoot`) cannot verify a username maps to non-zero. Place `USER` after the RUNs that need root.
6. **Memory limits: know the swap formula.** `-m 512m` alone allows swap = 2× memory, so the real ceiling is 1 GiB. Hard cap: set `--memory-swap` equal to `--memory` (`-m 512m --memory-swap 512m` = no swap).
7. **Cap logs at run time.** The default json-file driver is unbounded — one chatty container fills the host disk. `--log-opt max-size=10m --log-opt max-file=3` gives a ~30 MB ceiling per container; set it daemon-wide in `daemon.json` so nobody forgets.
8. **Gate startup on health, not on start.** Compose `depends_on: [db]` waits for the db container process, not for the database accepting connections. Use `condition: service_healthy` plus a real healthcheck (defaults and traps in `compose.md`).

## Exit Codes

Formula: a code above 128 means killed by signal `code − 128`.

| Code | Meaning | First move |
|------|---------|-----------|
| 125 | Docker daemon error (bad flag, missing image) | Read the run command, not the app |
| 126 | File found but not executable | `chmod +x`; or the entrypoint is a directory |
| 127 | Command not found | PATH wrong, or glibc binary on musl (Alpine) |
| 137 | SIGKILL (128+9) | Check `.State.OOMKilled`; also fired by stop-timeout expiry |
| 139 | SIGSEGV (128+11) | Native crash — suspect glibc/musl or architecture mismatch |
| 143 | SIGTERM (128+15) | Clean external stop — usually not a bug |

## Networking

- Publishing (`-p`) exists only for host and external access. Containers on the same user-defined network reach each other on ALL ports regardless of `-p` — and cannot be firewalled apart from each other by default.
- On Linux, published ports bypass ufw: Docker writes its own iptables chain ahead of ufw's rules. `-p 5432:5432` on an internet-facing box is a public database; use `-p 127.0.0.1:5432:5432` for local-only.
- `localhost` inside a container is the container. Service-to-service traffic uses the service name as hostname on a shared user-defined network.

## Disk Leaks

`/var/lib/docker` at 100% hangs the daemon itself — you cannot prune through a daemon that won't respond. Alert well before full; locate leaks with `docker system df -v`.

| Leak | Reclaim |
|------|---------|
| Dangling images | `docker image prune` |
| Build cache | `docker builder prune --keep-storage 10GB` |
| Stopped containers | `docker container prune`, or `--rm` at run time |
| Named volumes | `docker volume prune` — NOT touched by `system prune` without `--volumes`; destructive, confirm first |
| Orphan networks | `docker network prune` |

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Secrets via ENV, ARG, or COPY | All three persist in image history — deleting the file in a later layer does not remove the earlier layer | BuildKit `RUN --mount=type=secret`; runtime env or mounted files (`security.md`) |
| Mounting `/var/run/docker.sock` | Socket access = root on the host; any container escape is total | Dedicated proxy with a filtered API, or rethink the design |
| `ADD` for local files | Auto-extracts archives; URL downloads bypass the build cache | `COPY`; fetch URLs in a RUN with checksum verification |
| `docker logs` shows nothing | Only PID 1's stdout/stderr is captured | Log to stdout, or symlink the logfile to `/dev/stdout` |
| No shell in distroless/slim image | Nothing to `exec` into | `docker cp` files out, or attach a sidecar: `docker run --rm -it --network container:app nicolaka/netshoot` |
| `--privileged` to fix a permission error | Disables every isolation mechanism at once | Find the one capability or device needed (`security.md`) |
| Bind mount over an image path | Host dir replaces container contents; empty host dir = empty app dir | Named volume — it seeds from the image on first use |
| `restart: always` on dev boxes | Containers you stopped by hand come back after every host reboot | `unless-stopped` |

## Output Gates

Before emitting a Dockerfile or Compose file, verify:

- Base image pinned — tag at minimum, digest if this ships to prod?
- Dependency install layered before source copy?
- CMD/ENTRYPOINT in exec form?
- `USER` set with a numeric UID, or root explicitly justified?
- No secret reachable via ENV, ARG, or a COPYed file?
- `.dockerignore` present and excluding `.git` and dependency directories?
- Compose: healthcheck defined on every service something depends on?

## Where Experts Disagree

- **Alpine vs debian-slim.** Alpine is smaller but musl breaks prebuilt Python wheels and has DNS edge cases. Default: slim for Python/Node, Alpine for Go and static binaries; switch only when image size is a measured constraint.
- **Compose in production.** Legitimate for single-host deployments; the boundary is multi-host scheduling, rolling deploys, or autoscaling — those needs, not fashion, justify an orchestrator.
- **One process per container.** The default. Escape hatch: a process supervisor when the platform offers no sidecar mechanism — never as a convenience to avoid writing a second service.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/docker (install if the user confirms):
- `devops` — deployment pipelines
- `linux` — host system management
- `server` — server administration

## Feedback

- If useful, star it: https://clawic.com/skills/docker
- Latest version: https://clawic.com/skills/docker

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/docker.
