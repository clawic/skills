# Commands — Docker

The incident toolkit: commands that solve real problems, not the basics.

## Forensics on Dead Containers

```bash
docker logs --tail 100 app          # logs survive container death
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}' app
docker cp app:/var/log/app.log ./   # works on stopped containers
docker diff app                     # files changed vs image — spots rogue writes
docker commit app snap:debug        # freeze a broken container, rerun with a shell:
docker run --rm -it --entrypoint sh snap:debug
```

## Live Inspection

```bash
docker stats --no-stream            # point-in-time CPU/mem across containers
docker top app                      # processes without needing a shell inside
docker events --since 30m           # daemon view: OOM kills, health flips, restarts
docker update --memory 1g --memory-swap 1g app   # raise limits without restart
```

Detach from `docker attach` with Ctrl-P Ctrl-Q — Ctrl-C kills PID 1.

## Networking Debug

```bash
docker port app                                       # actual published mappings
docker network inspect mynet                          # members + IPs on a network
docker run --rm -it --network container:app nicolaka/netshoot   # tcpdump/dig/curl inside app's netns
```

## Images

```bash
docker history --no-trunc img       # audit layers for leaked secrets and size hogs
docker build --pull -t img .        # refresh the base tag; cache otherwise pins a stale base
docker buildx imagetools inspect img:tag   # digest + platforms without pulling
```

- `docker tag` copies nothing — retagging is free.
- `save`/`load` keeps layers and metadata; `export`/`import` flattens and LOSES ENTRYPOINT, CMD, and ENV.

## Compose

```bash
docker compose config               # final merged+interpolated file — first move when "compose ignores my setting"
docker compose up -d --wait         # block until healthchecks pass; the CI gate
docker compose down --remove-orphans   # renamed services otherwise leave zombie containers
docker compose down -v              # also deletes named volumes — destructive
```

## Cleanup

`docker system df -v` first — locate before deleting. Prune matrix: → SKILL.md Disk Leaks. Scope the prune to `build_cache_budget_gb`, and when `destructive_confirm` is true, name exactly what dies before running it.

Anything these commands establish that outlives the session has a home: a cause in `## Pain Points`, a machine or network fact in `## Environment`, a deployed digest in `deploys/<year>.md`, a host in the shared `~/Clawic/data/servers/servers.md` (`memory-template.md`). A forensic finding that stays in the scrollback gets rediscovered at the same cost next quarter.
