# CI — Building Images Where There Is No Yesterday

CI runners are amnesiac: every trick that makes local builds fast (warm cache, existing images) is absent. CI Docker work is cache strategy + tagging strategy + a security boundary decision. `ci_platform` in `config.yaml` decides the dialect of every example below; when it is unset, say which one you are assuming before writing a pipeline file.

**Before changing a pipeline**, read `## Stacks` and `## Registries` in `~/Clawic/data/docker/memory.md` — the platforms this project actually builds, the registry it pushes to, the mirror in front of Docker Hub and the retention policy on PR tags are recorded there (`registry.md` for the registry side).

## Cache in CI (the difference between 90s and 9min builds)

- Registry-backed cache is the portable answer:

```bash
docker buildx build \
  --cache-from type=registry,ref=myrepo/app:buildcache \
  --cache-to   type=registry,ref=myrepo/app:buildcache,mode=max \
  -t myrepo/app:$TAG --push .
```

- `mode=max` caches intermediate layers of ALL stages (multi-stage builds get almost nothing from the default `min`).
- GitHub Actions: `type=gha` cache is simpler than registry cache and scoped to the repo — but evicts at ~10 GB per repo; big matrices thrash it, registry cache doesn't.
- `--cache-from` does nothing without BuildKit/buildx, and nothing if the Dockerfile busts its own cache (layer order, images.md) — fix ordering before adding cache infrastructure.
- Lockfile discipline pays double in CI: `npm ci`/`pip install -r` layers cache only when the lockfile is in its own early COPY.

## Tagging Strategy

- Every CI build pushes an immutable identity tag: `app:<git-sha>` (short sha is fine). Movable tags (`latest`, `staging`, `prod`) are POINTERS you retarget, never the thing you deploy by.
- Deploy by digest or by sha-tag; promote by retagging the SAME digest (`docker buildx imagetools create -t app:prod app@sha256:...`) — promotion must never rebuild, or staging validated one artifact and prod runs another.
- PR builds: tag `app:pr-<num>`, and give the registry a cleanup policy for them, or they become the disk leak nobody owns.

## Multi-Arch (the laptop-is-arm64, prod-is-amd64 era)

```bash
docker buildx create --use   # once per runner
docker buildx build --platform linux/amd64,linux/arm64 -t app:$TAG --push .
```

- Cross-arch builds run through QEMU emulation: typically 3-10× slower for compiled languages — native runners per arch + `imagetools create` to stitch one manifest beat emulation at scale.
- The classic failure without this: image built on an arm64 laptop, `exec format error` on the amd64 server (debug.md, "works locally").
- `--load` (into the local daemon) supports single-platform only; multi-platform must `--push` to a registry.

## Docker-in-CI: the Boundary Decision

| Option | Reality | Use when |
|---|---|---|
| Runner's own daemon (socket) | Fast, shared cache — but jobs can see/kill each other's containers and the runner's; socket = root on runner | Trusted, internal repos |
| DinD service (dind) | Isolated daemon per job, no shared cache (pair with registry cache); requires privileged runner | Untrusted PRs on self-hosted infra |
| Rootless/daemonless builders (buildkit rootless, kaniko-style) | No privileged, no socket; some Dockerfile features (RUN --privileged) unavailable | Hardened/multi-tenant CI |

- Never expose the host socket to PR-triggered jobs from forks — that is remote root on your runner as a service.

## CI-Only Failure Modes

- Rate limits: anonymous Docker Hub pulls are throttled per-IP; a busy runner farm behind NAT hits limits at random-looking times. Authenticate pulls or mirror the handful of base images you use.
- `--pull` on builds (or a scheduled base refresh) — otherwise runners keep building on a base tag digest cached the day the runner was born (stale-base drift the laptop never sees).
- The context upload happens per-build with no local reuse: a fat context that costs 2s locally costs 30s × every job × every day (`.dockerignore`, images.md Size).
- Secrets: CI-injected env vars leak into images via ARG/ENV exactly like local ones — the BuildKit secret mount (security.md) is MORE important in CI, where "temporary debug echo" ends up in a pushed layer.

## Verify What You Shipped

- `docker buildx imagetools inspect app:$TAG` — digest + platforms without pulling; assert the platforms you meant to build.
- Smoke the artifact, not the source: run the PUSHED image (`docker run --rm app@$DIGEST --version` or your healthcheck) as a CI step — catches "builds fine, missing runtime dep" before deploy does.
- Record the digest in the build output/artifact metadata — it is your rollback key (production.md).

**When a released build ships**, write its row in `~/Clawic/data/docker/deploys/<year>.md`: date, service, host, digest, tag, rollback target, result (SKILL.md Rule 9, format in `memory-template.md`). **When the pipeline's shape changes** — platforms built, cache backend, the socket-boundary choice — update the service's row in `## Stacks` and, if the working pipeline file is worth re-reading, save it to `artifacts/<kebab-name>.md` with its `## Boxes` line in the same turn. Every CI credential is a pointer in those files (`env:REGISTRY_TOKEN`), never a value.
