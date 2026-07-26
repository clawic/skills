# Storage — Volumes, Mounts, and Data You Can't Lose

Decision rule first: **named volume for data the container owns** (databases, caches, uploads); **bind mount for data you edit on the host** (source code, config you tweak); **tmpfs for data that must not survive** (secrets at rest, scratch space).

**Before any backup, restore, or command that deletes a volume**, read `## Volumes` in `~/Clawic/data/docker/memory.md` (or `volumes.md` if `## Boxes` points there): what each volume holds, how it is backed up, and whether the restore has ever been tested. `down -v` against a volume whose row says `Restore tested: never` is a data-loss event with a countdown, not a cleanup.

## Named Volumes

- Seed-once semantics: a named volume copies the image's directory contents on FIRST use only — rebuilding the image never refreshes it. "My schema changes aren't appearing" after a rebuild = this; recreate the volume deliberately.
- `docker volume inspect <v>` shows the host path (`Mountpoint`) — direct host access works on Linux; on Desktop it lives inside the VM (go through a container instead).
- Anonymous volumes (a bare `VOLUME` in the Dockerfile, or `-v /path` with no name) accumulate invisibly and survive `docker rm` unless the container ran with `--rm`. Audit: `docker volume ls -f dangling=true`.
- `VOLUME` in a Dockerfile also FREEZES that path: any RUN after it writes to a throwaway layer that is discarded with no warning. Put `VOLUME` last, or omit it and declare at run time.

## Bind Mounts

- Mount over a populated image path and the image's files vanish for that run; an empty host dir = empty app dir. Named volumes don't have this trap (they seed).
- Permission denied = numeric UID mismatch between the container user and the host owner. Fix at build (`COPY --chown=10001:10001`, run as that UID); `chmod 777` is the finding, not the fix.
- Relative paths: in Compose, `./data` is relative to the compose FILE; with `docker run -v`, relative paths are not allowed at all (use `$(pwd)`).
- macOS/Windows: every bind mount crosses a VM boundary. Heavy I/O trees (`node_modules`, `.venv`, build caches, database files) belong in named volumes; bind-mount the source code around them: `-v ./:/app -v /app/node_modules`.
- SELinux hosts (Fedora/RHEL): add `:z` (shared) or `:Z` (private) or every access dies with EACCES regardless of UID.

## Backup and Restore (the part everyone improvises badly)

```bash
# Backup a named volume to a tarball (works everywhere, including Desktop)
docker run --rm -v pgdata:/from -v "$PWD":/to alpine tar czf /to/pgdata-$(date +%F).tar.gz -C /from .
# Restore into a fresh volume
docker volume create pgdata2
docker run --rm -v pgdata2:/to -v "$PWD":/from alpine tar xzf /from/pgdata-2026-07-23.tar.gz -C /to
```

- Databases: prefer the engine's own dump (`pg_dump`, `mysqldump`) over file-level copies of a RUNNING database — file copies of live data dirs restore corrupt.
- Moving data between hosts = the same tarball pattern + scp; there is no built-in volume migration.
- **A backup that has never been restored is a hypothesis.** Restore into a scratch volume, start the app against it, and time it. Schedule the drill as a `## Due` row (quarterly is the usual default) rather than trusting that it happened.

**Write after every volume event**: creation, a change of what it holds, a backup method, a backup run, and above all a restore — the row goes in `## Volumes` of `~/Clawic/data/docker/memory.md`, with `Restore tested` carrying a date and a measured duration or the literal word `never` (`memory-template.md`). Once the section passes ~15 volumes it splits to `volumes.md` with the same headings plus `## Restore Log`. The word `never` in a column is what turns an assumption into a visible risk; deleting it because it looks bad is how the assumption survives.

## tmpfs

- `--tmpfs /tmp` (or Compose `tmpfs:`) = RAM-backed, gone at stop, never touches disk — the right home for decrypted secrets and scratch files under `--read-only` (security.md).
- Size it: `--tmpfs /tmp:size=64m` — unsized tmpfs can eat RAM inside your memory limit and masquerade as an app leak.

## Drivers and the Filesystem Underneath

- Pin `overlay2` everywhere (default on modern engines); mixed drivers between hosts change rename/hardlink semantics and produce "works on my server" bugs.
- Container-layer writes are copy-on-write: the first write to a large image file copies the whole file. Databases writing in place suffer — another reason their data dirs belong on volumes (volumes bypass overlay entirely).
- NFS-backed volumes: fine for bulk artifacts; pathological for dependency trees (tens of thousands of small files → per-file latency dominates).

## Disk Accounting

- `docker system df -v` is the map: images, containers (their writable layers!), volumes, build cache — a "small" container with a huge writable layer means the app writes inside the container instead of a volume; find it with `docker diff <c>`.
- Log files are the stealth consumer — the json-file driver cap (SKILL.md rule 7) is a storage decision as much as a logging one.
- Prune matrix and the daemon-hang-at-100% warning: SKILL.md Disk Leaks.
