# Infrastructure — Networking, Volumes, Resources

## Networking

- `host.docker.internal` resolves out of the box on Docker Desktop; on Linux (Engine >=20.10) add `--add-host=host.docker.internal:host-gateway`.
- Container IPs change on every restart — never hardcode them; use service names or `--network-alias`.
- Bind rules: the app listens on `0.0.0.0` to be reachable at all; the HOST-side of `-p` decides exposure (`127.0.0.1:5432:5432` local-only). Firewall bypass and same-network reachability: → SKILL.md Networking.
- VPNs commonly lower the host MTU below the Docker network's 1500: symptoms are small requests fine, large payloads hang. Fix: set the network MTU to match the VPN interface.

## DNS

- The embedded resolver lives at 127.0.0.11 inside containers on user-defined networks; the default bridge has no name resolution at all.
- `--dns` replaces the resolver list; it does not append.
- A container with `--network=none` resolves nothing — including its own hostname.

## Volumes

- Anonymous volumes (from `VOLUME` in a Dockerfile) accumulate invisibly; `docker run --rm` cleans them, plain `rm` does not. Prefer named volumes.
- Bind-mount permission denied = UID mismatch between container user and host owner. Fix at build (`--chown` in COPY, matching numeric UID) rather than `chmod 777`.
- macOS/Windows bind mounts cross a VM boundary — heavy I/O dirs (`node_modules`, build caches) belong in named volumes, with the source code bind-mounted around them.
- Network-backed volumes (NFS) add per-file latency: fine for bulk data, pathological for dependency trees with tens of thousands of small files.

## Storage and Logs

- Log rotation and the per-container ceiling: → SKILL.md rule 7. Set it in `daemon.json` so it applies to containers nobody remembered to flag.
- `/var/lib/docker` full hangs the daemon (→ SKILL.md Disk Leaks); monitor that filesystem specifically — root having space is no guarantee.
- Mixed storage drivers between dev and prod hosts produce different filesystem semantics (rename, hardlinks); pin `overlay2` everywhere it's supported.

## Resources

- No `--memory` limit = the container can take all host RAM and invite the kernel OOM killer to kill something else. Swap interaction and the hard-cap formula: → SKILL.md rule 6.
- `--cpus=0.5` is a ceiling, not a reservation — under contention the container still competes below it.
- JVMs older than JDK 8u191/10 size their heap from HOST memory, ignoring cgroup limits — a 512 MB container with 32 GB host RAM OOMs on startup. Modern JVMs respect limits by default; on old ones set `-XX:+UseContainerSupport` or `-Xmx` explicitly.
- Diagnosis order for a container that "randomly dies": `docker inspect -f '{{.State.OOMKilled}}'` → `docker events --since 1h` → host `dmesg | grep -i oom`. The host kernel log catches OOM kills the daemon never attributes.
