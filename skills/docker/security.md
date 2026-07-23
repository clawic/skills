# Security Traps

## User

- Container runs as root by default, security scanners flag it
- `USER` directive after a `RUN` that needs root = build fails
- User in container with UID 1000 = may be a different user on the host, confusing
- `--user` at runtime overrides the Dockerfile's USER, but file permissions remain

## Secrets

- `ENV SECRET=x` visible in `docker history` and `docker inspect`
- `ARG` for secrets is also visible in history, not secure
- `COPY secrets.txt` baked into a layer; even if you delete it later, it's in the previous layer
- `--env-file` is safe at runtime but the file must be protected on the host

## BuildKit Secrets

- `RUN --mount=type=secret` not available without DOCKER_BUILDKIT=1
- Secret mount only available in that RUN, it doesn't persist
- Secret ID must match exactly, a typo = build fails without a clear message
- Secret not available in stages that don't mount it explicitly

## Image Scanning

- Vulnerabilities in the base image are inherited, update the base regularly
- Scanning in CI but not in the registry = vulnerable images in production
- CVE "fixed" in a package but the base image not updated = still vulnerable
- Distroless images are hard to scan, fewer CVEs reported, not fewer bugs

## Runtime

- `--privileged` = full access to host devices, kernel modules, etc.
- `--cap-add SYS_ADMIN` is almost as bad as privileged, avoid it
- `-v /:/host` mounts the host root = game over if the container is compromised
- `--pid=host` lets you see/kill host processes from the container

## Network

- A container on the bridge network can reach the metadata service (169.254.x.x)
- Without `--network=none`, the container has network access by default
- Published ports without a firewall = public to the internet
- A container can make requests to other containers on the same network, no isolation

## Supply Chain

- A base image from a public registry can be malicious, verify the publisher
- The `latest` tag can be hijacked, use a digest for critical images
- Dependencies downloaded during build can change, lock files + verified mirrors
