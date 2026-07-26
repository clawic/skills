# Networking — Reachability, DNS, and the Firewall Truth

Mental model: every container has its own network namespace; a "network" is a virtual switch with an embedded DNS server; `-p` is a NAT rule from the host into one namespace. Most "networking bugs" are one of these three pieces misassigned.

**Before diagnosing reachability on a machine you have seen before**, read `## Environment` in `~/Clawic/data/docker/memory.md`: the VPN MTU, the corporate CA, the daemon DNS override, the address pool chosen to avoid a collision, and which host ports are already taken by non-Docker services are recorded there. Those five facts account for most of the failures below and none of them are discoverable from the container.

## Reachability Matrix

| From → To | How | Gotcha |
|---|---|---|
| Host → container | `-p hostPort:containerPort` | App must bind `0.0.0.0` inside; `127.0.0.1` inside = unreachable no matter what you publish |
| Container → host | `host.docker.internal` | Linux Engine needs `--add-host=host.docker.internal:host-gateway`; Desktop resolves it natively |
| Container → container (same user-defined network) | service/container name, ANY port | `-p` irrelevant here; all ports open between them by default |
| Container → container (default bridge) | IP only | No DNS on the default bridge — the classic "ping by name fails" |
| Container → internet | NAT via host | Corporate proxy/CA issues surface here (below) |
| Outside → container | `-p` + host firewall/cloud rules | Linux: Docker bypasses ufw (below) |

## The Firewall Truth (Linux)

- Docker programs its own iptables chains (`DOCKER`) evaluated BEFORE ufw/firewalld rules: `-p 5432:5432` on an internet-facing host is a public database even with ufw "deny all".
- Local-only publishing is the fix that always works: `-p 127.0.0.1:5432:5432`.
- Fleet-level: set `"iptables": false` only if you fully own the firewall config — it breaks container internet access until you replicate the NAT rules yourself. Most teams should publish on localhost + reverse-proxy instead.
- Cloud metadata endpoint `169.254.169.254` is reachable from default networking — SSRF in a container becomes credential theft; block it host-level (security.md).

## DNS

- Embedded resolver = `127.0.0.11` inside user-defined networks; it forwards non-container names to the daemon's DNS config.
- `--dns 8.8.8.8` sets the UPSTREAM of the embedded resolver and REPLACES the list (does not append); container-name resolution keeps working. The real trap is a daemon-level `"dns"` misconfig, which breaks external lookups for every container at once.
- Split-horizon/VPN DNS: containers don't inherit the host's resolver magic (systemd-resolved, VPN split DNS). Symptom: host resolves `internal.corp`, container doesn't. Fix: daemon.json `"dns": ["<corp resolver>"]` or per-run `--dns`.
- `getent hosts <name>` is the portable in-container test (exists in musl and glibc); `nslookup`/`dig` usually aren't installed.

## MTU (the VPN hang)

- Symptom signature: TLS handshakes or small requests fine, large POSTs/downloads hang forever. Cause: VPN lowers host MTU below the Docker network's default 1500 and PMTU discovery is blocked.
- Check: `ip link` on the host (VPN interface MTU, often 1400 or 1380) vs `docker network inspect -f '{{index .Options "com.docker.network.driver.mtu"}}' bridge`.
- Fix: create networks with the lower MTU (`docker network create -o com.docker.network.driver.mtu=1400 mynet`) or daemon.json `"mtu": 1400`; Compose: `driver_opts` on the network.

## Proxies and Corporate CAs

- Build-time and runtime are separate: `HTTP_PROXY` as build ARG for RUN steps; daemon proxy config (`~/.docker/config.json` proxies block) injects into containers at run.
- Custom CA: the container trusts only ITS OWN CA store — mount or COPY the corp root CA and run the distro's update command (`update-ca-certificates` on Debian/Alpine) or set the language-level override (`NODE_EXTRA_CA_CERTS`, `REQUESTS_CA_BUNDLE`, `SSL_CERT_FILE`).
- TLS errors only in-container, browser fine on host = 90% corporate MITM proxy + missing CA.

## Ports

- "Port is already allocated" → `docker ps --format '{{.Names}} {{.Ports}}' | grep <port>` finds the holder; also check non-Docker listeners with `ss -ltnp`.
- `-p 8080:80` order is HOST:CONTAINER — reversed order "works" until it doesn't; a connection-refused on the port you expected is often this.
- `--network host` (Linux only, real effect): no publishing, no isolation, container ports ARE host ports — port conflicts become app-level crashes; Desktop's host mode is emulated per-port.
- Ephemeral publishing `-p 80` (no host part) picks a random host port — read it with `docker port`.

## IPv6

- Disabled by default on the Docker bridge; a host with broken IPv6 + an image resolving AAAA first can show "network unreachable" for hosts that "work in the browser". Quick test: force IPv4 in the app or add `"ipv6": false` expectations; enabling real v6 requires daemon `"ipv6": true` + a ULA `"fixed-cidr-v6"`.

## Inspection Toolkit

```bash
docker network ls && docker network inspect <net>        # membership, subnets, MTU option
docker run --rm -it --network container:<c> nicolaka/netshoot   # tcpdump, dig, curl inside <c>'s namespace
docker exec <c> getent hosts <name>                      # portable resolution test
docker exec <c> sh -c 'ss -ltn || netstat -ltn'          # who listens where
```

**Write the network facts that are properties of the machine, not of the bug**: the VPN's MTU and the value the networks were created with, the corporate CA and where it is mounted, a daemon-level DNS or address-pool override, a host port permanently held by a non-Docker service. One line each in `## Environment` of `~/Clawic/data/docker/memory.md`, in the same turn you establish them (`memory-template.md`). Anything that only makes sense as a whole file — a `daemon.json`, a working proxy and CA setup — is an `artifacts/` file with its `## Boxes` line. These are the facts that are invisible from inside a container and expensive to rediscover.
