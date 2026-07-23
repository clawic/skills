# Infrastructure Traps

## Networking

- `localhost` inside a container is the container, not the host, use `host.docker.internal`
- `0.0.0.0` bind is needed for the container to be reachable, `127.0.0.1` is only local to the container
- `-p 5432:5432` without an IP = binds to all interfaces = public if there's no firewall
- Container restart changes the IP, use network aliases, not hardcoded IPs

## DNS

- Default DNS is 127.0.0.11 internal, it doesn't use the host's /etc/resolv.conf
- `--dns` is a full override, it doesn't add, it replaces
- DNS caching in the daemon, external DNS changes take time to propagate
- A container without a network has no DNS, not even localhost resolves

## Volumes

- Anonymous volume (`VOLUME` in the Dockerfile) accumulates without limit, never auto-deleted
- `docker system prune` does NOT delete volumes, it needs an explicit `--volumes`
- Bind mount permissions: container user vs host user — mismatch = permission denied
- NFS volumes with latency = horrible performance, especially for node_modules

## Storage Driver

- `overlay2` is the default but overlayfs on an old kernel = subtle bugs
- Different storage driver between dev/prod = different behavior
- Logs without a limit grow infinitely, `--log-opt max-size=10m`
- `/var/lib/docker` full = daemon hangs, monitoring is essential

## Resources

- Without a `--memory` limit = the container can use all the RAM and trigger the OOM killer
- `--memory` without `--memory-swap` = swap = 2x memory, which can be a lot
- `--cpus=0.5` is a limit, not a reservation, other containers can use it
- Java in a container without `-XX:+UseContainerSupport` doesn't see the correct limit

## Security

- `--privileged` disables ALL security, almost never needed
- Granular `--cap-add` is better than privileged, only what you need
- Root in the container can be root on the host, use user namespaces to avoid it
- Secrets in env vars are visible with `docker inspect`, use secrets/mounts
