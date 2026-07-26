# Debugging — Symptom to Cause in Minutes

Diagnosis is where containers eat hours. Work symptom-first; each chain is ordered by probability, and every step is a check, not a guess.

**Before the first command**, read `## Pain Points` and `## Environment` in `~/Clawic/data/docker/memory.md` and open any `artifacts/runbook-*.md` its `## Boxes` index names for this symptom. Half of all repeat incidents are the same incident, and the runbook is faster than the chain below. Runtime-specific behavior (VM ceiling, rootless, Podman) is in `runtimes.md`; language runtime behavior under a cgroup is in `languages.md`.

## The Universal First Three

1. `docker logs --tail 100 <c>` — works on dead containers; read the LAST lines first (the crash), then scroll up for the cause.
2. `docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}} {{.State.Restarting}}' <c>` — exit code decodes via the table in SKILL.md.
3. `docker events --since 1h` — the daemon's view: OOM kills, health flips, restart storms that logs never mention.

## Crash Loop (restarting forever)

1. `docker inspect -f '{{.RestartCount}} {{.State.ExitCode}}'` — code 137 → OOM chain below; 1 or app-specific → app config.
2. Freeze one iteration: `docker update --restart=no <c>`, let it die, read logs in peace.
3. Config/env suspicion: `docker inspect -f '{{json .Config.Env}}' <c> | tr ',' '\n'` — compare against what the app expects; the #1 cause is an env var that exists in your shell but was never passed.
4. Still opaque: `docker commit <c> snap && docker run --rm -it --entrypoint sh snap` — walk the filesystem the app actually sees.

## OOM (exit 137, OOMKilled=true)

1. Confirm attribution: host `dmesg -T | grep -i oom` — the kernel kills by cgroup; sometimes it killed a child process, and PID 1 exited on its own.
2. Real usage before death: `docker stats --no-stream` on a replica under load; remember the swap doubling (SKILL.md rule 6) when reasoning about the ceiling.
3. Language runtimes need their own cap BELOW the container's: JVM `-Xmx` at ~75% of `-m` (and JVMs older than 8u191/10 size the heap from HOST memory, ignoring the cgroup — a 512 MB container on a 32 GB host OOMs at startup unless `-Xmx` is explicit); Node `--max-old-space-size` in MB at ~75-80%; Python has no cap — profile or bound the workload.
4. Limit honest but breaches spiky: the kernel counts page cache written by the container against the cgroup — heavy file I/O "leaks" that isn't a leak; raise the limit or throttle writes.

## "Works Locally, Fails in CI/Prod"

Check in this order; each is a one-minute test:

| Difference | Check |
|---|---|
| Architecture (arm64 laptop vs amd64 runner) | `docker inspect -f '{{.Architecture}}' <img>` both sides; exit 139/`exec format error` = this |
| Image staleness | Compare `docker images --digests` — CI pulled a different digest of the same tag |
| Env vars present locally, absent in CI | `docker inspect Config.Env` diff |
| Bind mount only exists locally | grep compose/run flags for `./` paths |
| Case-sensitive filesystem (Linux) vs insensitive (macOS) | imports/paths that differ only by case |
| Missing `--platform` on a multi-arch base | build logs show which variant was pulled |

## Port Unreachable

1. Is anything listening? `docker exec <c> sh -c 'command -v ss >/dev/null && ss -ltn || netstat -ltn'` — if the app listens on `127.0.0.1`, publishing can't help; bind `0.0.0.0` (SKILL.md Quick Reference).
2. Is it published? `docker port <c>` — shows the REAL mappings, not what you meant to type.
3. No shell in the image: `docker run --rm -it --network container:<c> nicolaka/netshoot ss -ltn` — same netns, full toolbox.
4. Reachable from host but not from outside: host firewall or cloud security group; on Linux remember published ports bypass ufw (networking.md).
5. Reachable but resets under load: check `docker events` for the container restarting behind you — you are testing a different instance each time.

## DNS / Service Discovery Failure

1. Which network? `docker inspect -f '{{json .NetworkSettings.Networks}}' <c>` — default bridge = no DNS at all, the most common cause.
2. Test resolution from inside the failing container, not from the host: `docker exec <c> getent hosts <service>` (getent exists where nslookup doesn't).
3. Wrong name: Compose DNS uses the SERVICE name, not the container name (`compose.md`).
4. Resolves but connection refused: dependency started but app inside it hasn't bound yet — healthcheck gating (SKILL.md rule 8).
5. External DNS broken, internal fine: corporate VPN/DNS — test `docker run --rm busybox nslookup example.com`; fix daemon `dns` in daemon.json (networking.md).

## Build Fails

- `apt-get install` 404s → stale package index from a cached earlier layer (SKILL.md rule 3).
- `COPY failed: file not found` → file exists on disk but is excluded by `.dockerignore`, or you're building from a different context dir than you think: `docker build` context is the LAST argument.
- Hash mismatch/`failed to compute cache key` → stale builder state: `docker builder prune`, retry once, then look for a file changing mid-build (generated files in context).
- Works with `--no-cache` only → a cached layer holds stale state; find the earliest wrong layer with `docker history` and bust that line specifically (images.md).
- Hangs at `load build context` → giant context; fix `.dockerignore` first (images.md Size).

## Permission Denied

- On a bind mount → UID mismatch; fix with matching numeric UID at build, never `chmod 777` (storage.md).
- On the app's own directories after `--user` override → files baked as root; `COPY --chown` at build (security.md).
- `permission denied ... docker.sock` on the host → user not in `docker` group; remember that membership is root-equivalent (security.md).
- Rootless runtime: ports <1024 and some mount types fail by design — check `runtime_flavor` in config before chasing ghosts.

## Slow (build or runtime)

- Slow build, cold cache every time → layer order or CI without a cache source (`ci.md`).
- Slow runtime I/O on macOS/Windows bind mounts → VM boundary; move heavy dirs to named volumes (storage.md).
- Slow network, large payloads only → MTU mismatch under VPN (networking.md).
- Slow everything, host loaded → another container without limits is starving the box: `docker stats` sorted by memory, then add limits (production.md).

## When You Are Truly Stuck

Reproduce minimal: `docker run --rm -it <image> sh` with ZERO flags, then re-add your real flags one at a time — the flag that breaks it names the subsystem, and the file above to open next.

## Write Down What It Was

A cause that took more than a couple of minutes to find is worth more than the fix. Three destinations (`memory-template.md`):

- **One line in `## Pain Points`** of `~/Clawic/data/docker/memory.md`: date, symptom, actual cause, what changed. This is what stops the next session from re-walking the chain.
- **A fact that will change future decisions** — the VM's memory ceiling, the VPN MTU, a corporate CA, a host port already taken, an SELinux relabel requirement — goes in `## Environment` instead, because it applies to everything, not just this incident.
- **The second time the same failure appears**, it stops being a note and becomes `artifacts/runbook-<symptom>.md`: the ordered checks, the fix, and the digest to roll back to. Add its `## Boxes` line with a read condition naming the symptom, so the next session opens it before the chain above rather than after.
