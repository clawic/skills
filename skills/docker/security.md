# Security — Hardening and Traps

## User

- Containers run as root unless the image says otherwise; numeric-UID rule and placement: → SKILL.md rule 5.
- `--user` at runtime overrides the Dockerfile's `USER`, but files baked with root ownership stay root-owned — the app then can't write its own directories. Bake ownership at build time (`COPY --chown`).
- UID 1000 in the container is whatever user owns UID 1000 on the host for bind mounts — match numeric IDs, not names.

## Secrets

- ENV, ARG, and COPYed files all persist in image history (→ SKILL.md Traps). The safe build-time pattern:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

- The secret exists only during that RUN and only in stages that mount it; a mismatched `id` fails the build with an unhelpful message — check the id first.
- `--env-file` at runtime keeps secrets out of the image, but `docker inspect` shows them to anyone with docker access, and docker access is root-equivalent anyway.

## Runtime Hardening

Default posture for a production service — remove pieces only when something breaks:

```bash
docker run --read-only --tmpfs /tmp \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  -m 512m --memory-swap 512m image
```

- `--cap-drop ALL` then add back the one or two capabilities the app actually needs beats guessing which to drop.
- `--privileged` and its near-equivalents (`--cap-add SYS_ADMIN`, `--pid=host`, `-v /:/host`, mounting the docker socket) each hand over the host — treat any of them in a config as a finding, not a style choice.

## Network Exposure

- A container with default networking can reach cloud metadata services (169.254.169.254) — an SSRF inside the container becomes credential theft. Block it at the host firewall or use per-container network policy.
- Same-network containers reach each other on every port (→ SKILL.md Networking) — segment by putting each trust boundary on its own network; a compromised frontend shouldn't see the admin service.
- Published ports on Linux bypass ufw (→ SKILL.md Networking).

## Supply Chain

- Digest-pin production images (→ SKILL.md rule 1) — a tag, including `latest`, can be repointed at a malicious image after you vetted it; a digest cannot.
- Rebuild regularly even without code changes: CVE fixes arrive in base images, and a base pinned for six months carries six months of known holes.
- Scan at two points — CI (blocks bad builds) AND the registry (catches CVEs published after the build). CI-only scanning approves images that rot in place.
- Distroless reports fewer CVEs partly because scanners have less to match against — smaller attack surface is real, "zero findings" is not proof of zero flaws.
