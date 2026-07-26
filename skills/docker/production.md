# Production — Running Containers on a Host You're Responsible For

Scope: single-host and Compose-level production (the majority of real deployments). Multi-host scheduling, rolling deploys across nodes, autoscaling → that is the orchestrator boundary (SKILL.md, Where Experts Disagree).

**Before any deploy, rollback or host change**, read `~/Clawic/data/servers/servers.md` (which hosts exist and what they run), `## Environment` in `~/Clawic/data/docker/memory.md`, and `deploys/<year>.md` if `## Boxes` points there — the digest you would roll back to lives in that file and nowhere else. **Check `## Due`** against today's date and state any overdue prune, base rebuild, restore drill or reboot drill in one line.

## The daemon.json Canon

Set fleet defaults once in `/etc/docker/daemon.json` so no forgotten flag can hurt you:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "storage-driver": "overlay2",
  "live-restore": true,
  "default-address-pools": [ { "base": "172.80.0.0/16", "size": 24 } ]
}
```

- `live-restore: true` — containers survive a daemon restart/upgrade; without it every `systemctl restart docker` is an outage.
- `default-address-pools` — corporate networks collide with Docker's default 172.17-172.31 ranges; a pool you chose beats debugging "VPN broke when compose started".
- Log caps daemon-wide (SKILL.md rule 7): per-run flags protect one container; daemon.json protects the host.
- Changes need a daemon restart; with live-restore the containers keep running through it.

**A `daemon.json` you wrote is an artifact**: save it to `~/Clawic/data/docker/artifacts/daemon-json-<host>.md` with the reasoning behind each key and its `## Boxes` line in the same turn (`memory-template.md`). Deriving an address pool that does not collide with the corporate VPN once is work; deriving it twice is an outage in between.

## Restart Policies (know the semantics before an outage)

| Policy | Behavior | Use |
|---|---|---|
| `no` | Never restarts | Debugging (freeze crash loops, debug.md) |
| `on-failure[:N]` | Restarts on non-zero exit, max N tries; NOT after daemon restart | Batch jobs |
| `always` | A manual `docker stop` sticks only until the daemon restarts — then the container resurrects | The surprise on reboot: hand-stopped containers come back |
| `unless-stopped` | Like always, but a manual stop is permanent across reboots | The right default for services |

- Restart backoff is exponential starting at 100ms and doubling, capped at 1 minute — a crash-looping service hammers itself far less than a naive systemd unit, but `RestartCount` climbing is your alarm signal.

## Deploying Updates Without an Orchestrator

- Standard: `docker compose pull && docker compose up -d` — recreates only services whose image or config changed; healthcheck-gated dependencies (rule 8) keep the order sane.
- Zero-downtime on one host is a reverse-proxy pattern, not a Docker feature: run the proxy (caddy/traefik/nginx) as the stable front, `up -d --scale app=2` the new alongside the old, let health gates admit it, then retire the old. If you need this weekly, that is the orchestrator signal.
- Auto-updaters (watchtower-style) trade control for convenience: they redeploy whatever the tag now points at — combine with digest pinning and they do nothing; combine with mutable tags and they deploy unvetted images at 3am. Choose explicitly.
- Rollback = deploy the previous digest. Which is only possible if you RECORDED it: tag every deploy (`myapp:2026-07-23-a1b2c3`) and log `docker inspect -f '{{index .RepoDigests 0}}'` at deploy time. **Write the row to `~/Clawic/data/docker/deploys/<year>.md` in the same turn** — date, service, host, digest, tag, rollback target, result (SKILL.md Rule 9, format in `memory-template.md`). A deploy whose digest lives only in a terminal scrollback has no rollback, only a rebuild.

## Monitoring the Right Things

- Host: `/var/lib/docker` filesystem usage (the daemon hangs at 100% — SKILL.md Disk Leaks), plus RAM and load.
- Per-container: restart count (`docker inspect -f '{{.RestartCount}}'`), health status, and memory vs limit (`docker stats`). A container at 95% of its limit is one allocation from an OOM loop.
- `docker events` is your flight recorder — ship it (`docker events --format '{{json .}}' >> /var/log/docker-events.jsonl &` as a systemd unit) so post-incident you can answer "what restarted when".
- Healthchecks feed `docker ps` STATUS and Compose gating, but nothing pages you — wire health flips from events into whatever alerts humans.

## Resource Stanzas (Compose v2, non-Swarm)

```yaml
services:
  app:
    mem_limit: 512m
    memswap_limit: 512m      # equal = no swap (SKILL.md rule 6)
    cpus: "1.5"
    pids_limit: 256           # fork-bomb ceiling
    restart: unless-stopped
    logging: { options: { max-size: "10m", max-file: "3" } }
```

- `cpus` is a ceiling, not a reservation — under contention the container still competes for its share below the cap.
- `deploy.resources` only applies under Swarm or `docker compose --compatibility`; the flat keys above always work — the silent-no-op difference bites during load tests.
- Every service gets a memory limit. No exceptions: one unbounded container invites the kernel to kill a random neighbor (infrastructure history's most misattributed outage).

## Host Hygiene

- One responsibility per host where you can: build hosts (cache-heavy, disk churn) separate from serve hosts (stability) — build cache pressure and prod uptime fight over the same disk.
- Prune ON A SCHEDULE, scoped: `docker image prune -af --filter until=168h` + `docker builder prune -af --keep-storage 10GB` weekly via cron/systemd-timer. Never schedule `system prune --volumes` — that is how backups get tested.
- Time drift inside containers follows the host (containers share the kernel clock): NTP on the host is a container-correctness requirement (TLS and JWT validation fail weirdly when it drifts).
- Reboot drill quarterly: `unless-stopped` policies + healthcheck gating should bring the whole host back with zero hands. If it doesn't, you found the incident before it found you.

Every cadence on this page — the weekly prune, the base-image rebuild and rescan, the volume restore drill, the reboot drill — is a row in the `## Due` table of `~/Clawic/data/docker/memory.md` with its last-run date. A maintenance schedule with no recorded last run gets skipped for two quarters and nobody notices until the disk does. Record the reboot drill's measured recovery time and what was still wrong in `deploys/<year>.md` under `## Drills`.

## Production Gate (before calling a container "deployed")

- Digest recorded for rollback?
- Memory + swap limit, pids_limit, log cap set?
- `restart: unless-stopped`, healthcheck present, dependencies health-gated?
- Runs as non-root UID; hardening flags from security.md applied?
- Its data on a named volume, and that volume in the backup routine (storage.md)?
- Published ports bound to 127.0.0.1 unless the port is meant for the world (networking.md)?
- Host row present in `~/Clawic/data/servers/servers.md` with `docker host` in `Role`, deploy row with its digest in `deploys/<year>.md`, and any new box indexed in `## Boxes` — all in this same turn (`memory-template.md`)?
