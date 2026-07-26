# Security — Hardening and Traps

**Before hardening anything**, read `## Stacks` in `~/Clawic/data/docker/memory.md` and any `artifacts/` file its `## Boxes` index names for this service: the capability that had to be added back, the CVE accepted with a reason, and the base-image decision are all things somebody already paid for once. `hardening_profile` in `config.yaml` decides whether the flags below are emitted by default or only on request.

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
- Same-network containers reach each other on every port (→ `networking.md`, Reachability Matrix) — segment by putting each trust boundary on its own network; a compromised frontend shouldn't see the admin service.
- Published ports on Linux bypass ufw, so `-p 5432:5432` on an internet-facing host is a public database even under "deny all" (→ `networking.md`, The Firewall Truth). Bind to `127.0.0.1` unless the port is meant for the world.

## Supply Chain

- Digest-pin production images (→ SKILL.md rule 1) — a tag, including `latest`, can be repointed at a malicious image after you vetted it; a digest cannot.
- Rebuild regularly even without code changes: CVE fixes arrive in base images, and a base pinned for six months carries six months of known holes.
- Scan at two points — CI (blocks bad builds) AND the registry (catches CVEs published after the build). CI-only scanning approves images that rot in place.
- Distroless reports fewer CVEs partly because scanners have less to match against — smaller attack surface is real, "zero findings" is not proof of zero flaws.

## What Gets Written Down

Hardening decisions decay the moment they leave the session. Three destinations, all in `memory-template.md`:

- **The hardened run/compose stanza or Dockerfile that finally worked**, including every capability that had to be added back and why it was needed → `~/Clawic/data/docker/artifacts/<kebab-name>.md`, with its `## Boxes` line in the same turn. `--cap-drop ALL` then re-adding one capability is a diagnostic session; nobody should run it twice for the same service.
- **A CVE accepted with a reason, or a scanner exception** → `artifacts/`, with the date, the finding id, the reason, and the condition under which it must be revisited. An exception with no revisit condition is a permanent hole with a note attached.
- **The rebuild-and-rescan cadence** → a `## Due` row. A base pinned six months ago carries six months of published CVEs, and only a dated schedule catches that.

**Secrets never land in any of it.** Everything under `~/Clawic/data/` — including files you create and text the user pastes for safekeeping — stores the pointer, not the value: `env:REGISTRY_TOKEN`, `keychain:ghcr-push`, `file:~/.docker/config.json`. Strip before writing, and say in one line that you did.
