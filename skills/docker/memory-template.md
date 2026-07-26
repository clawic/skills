# Working File Templates — Docker

Read this file only when WRITING. `config.yaml` is what the user **declared**; `memory.md` and everything it indexes is what you **observed** or produced. An observation never overwrites a declaration.

## Where each thing goes

| Data | Home | How it grows |
|---|---|---|
| Declared preferences — Configuration table keys and preference areas alike | `~/Clawic/data/docker/config.yaml` | Key by key, read-modify-write |
| Environment facts, stacks, volumes, registries, pain points, due dates, box index | `~/Clawic/data/docker/memory.md` | Rewritten in place; stays small |
| Docker hosts and the machines containers run on | `~/Clawic/data/servers/servers.md` (**shared**) | One row per host, every provider in one inventory |
| Stacks and the images they build — base, platform, pin, registry | `## Stacks` in `memory.md`; `~/Clawic/data/docker/stacks.md` once it outgrows the section | One row per image or service |
| Named volumes: what is in them, backup method, last restore test | `## Volumes` in `memory.md`; `~/Clawic/data/docker/volumes.md` once it outgrows the section | One row per volume |
| Deploys and the digest that rolls each one back | `~/Clawic/data/docker/deploys/<year>.md` | Append-only, cut by year |
| Things you produced that get re-read — a Dockerfile or compose file that finally worked, a `daemon.json`, a runbook, a base-image or hardening decision | `~/Clawic/data/docker/artifacts/<kebab-name>.md` | Born as its own file, from the first one |
| **Anything durable this table does not name** | `~/Clawic/data/docker/<plural-noun>.md`, or `artifacts/<kebab-name>.md` if it is a long text read whole | Name the file after what it holds, never after when it was made; add its `## Boxes` line in the same turn |
| Credentials of any kind | Nowhere under `~/Clawic/data/` | Pointer only — see Secrets |

Deciding where something unnamed goes, in this order: (1) would another skill want to read it — a host, a person, a project, a domain? Then it belongs in the shared box, not here. (2) Is it a text read whole when its subject comes up — a procedure, a config that took work to derive, a decision with its reasoning? Then `artifacts/`, its own file from the first one. (3) Is it one more row of something that accumulates? Then a section of `memory.md` until the split threshold.

## When to write

No permission needed, no announcement beyond one line.

| It happened | Write |
|---|---|
| A Docker host was provisioned, rebuilt, migrated or retired | Its row in `servers.md` (shared) |
| A stack was containerized, or its base image / platform / pin changed | `## Stacks` |
| A named volume was created, backed up, restored, or its restore was timed | `## Volumes` |
| An image was deployed | A row in `deploys/<year>.md` with its digest (SKILL.md Rule 9) |
| A rollback happened | The same row, plus what it rolled back to and why |
| An environment fact cost effort to find — VM ceiling, VPN MTU, corporate CA, registry mirror, a host port already taken, an SELinux relabel | `## Environment` |
| A registry was added, a credential helper configured, or a rate limit hit | `## Registries` |
| A failure's cause was not obvious, or the same failure appeared twice | `## Pain Points`; the second occurrence earns a runbook in `artifacts/` |
| A Dockerfile, compose file or `daemon.json` finally worked | `artifacts/` |
| A base-image, hardening, or orchestrator-boundary decision was made | `artifacts/`, with what was rejected and why |
| A prune, rebuild-and-rescan, restore drill or reboot drill was scheduled or run | `## Due` |
| The user declared a preference | Its key in `config.yaml` |

## Start flat, split only when it hurts

Everything except artifacts, deploy records and the shared inventory begins inside `memory.md`. Splitting is a procedure, not a suggestion:

1. Before appending to a section, count its entries.
2. If the append would take it past **~15 entries or ~40 lines of real content** — scaffolding, headings and comments do not count — then, in the same turn: create the new file in `~/Clawic/data/docker/`, move the whole section into it, **delete the section from `memory.md`**, add its line to `## Boxes`, and append the new entry to the new file.
3. Keep the headings identical on both sides of the move, so the split is a copy-paste and never a rewrite.
4. Never leave a copy behind. If the same data ever appears in both places, the extracted file wins and the `memory.md` copy is deleted.

Artifacts are the exception: a runbook, a working compose file or a decision is born as its own file whatever its size, because it is read whole and only when its subject comes up.

## Secrets

Nothing under `~/Clawic/data/` ever holds a secret value — not the files named here, not files you create, not text the user pastes in and asks you to keep. A pasted Dockerfile, compose file, `.env`, `daemon.json` or CI log is the densest source of secrets there is: strip each value **before** writing and leave its pointer in place, in this shape: `<kind>:<locator>`.

`env:REGISTRY_TOKEN` · `keychain:ghcr-push` · `1password:Work/Registry/ci` · `bitwarden:CI/dockerhub` · `vault:secret/ci/registry` · `file:~/.docker/config.json` · `file:~/.ssh/id_ed25519`

In a text, the pointer goes where the value was: `POSTGRES_PASSWORD: <env:POSTGRES_PASSWORD>`. Say in one line that you did it.

In this domain — **not secrets, keep them**: image names and tags, image and layer digests, registry hostnames and namespaces, container and service names, volume and network names, published ports, UIDs and GIDs, base-image families, platform strings, host names, CVE ids, environment *variable names*.

**Secrets, strip them**: registry passwords and access tokens, `~/.docker/config.json` auth blobs, `.npmrc`/`.pypirc` tokens, database passwords and connection strings that carry one, TLS private keys and their passphrases, SSH private keys, cloud access keys mounted into build steps, cosign private keys, webhook and pager tokens, any `--build-arg` or `--secret` value the user pastes.

**Contents:** [config.yaml](#configyaml) · [memory.md](#memorymd) · [shared servers inventory](#shared-servers-inventory) · [artifacts/](#artifacts) · [deploys/](#deploys) · [split-out files](#split-out-files)

## config.yaml

Keys come from the Configuration table in `SKILL.md`, plus free-form keys nested under a preference area. Write a key only when the user states the preference.

**Writing is read-modify-write**: load the existing file, set or replace only the key just declared, keep every other key byte for byte. Never emit a `config.yaml` from this template — the template shows shape, not content. Create `~/Clawic/data/docker/` if it does not exist.

```yaml
runtime_flavor: orbstack
default_registry: ghcr.io
base_image_family: debian-slim
default_platform: arm64
pin_policy: digest-in-prod
hardening_profile: hardened
ci_platform: github-actions
build_cache_budget_gb: 20
destructive_confirm: true

# Preference areas — free-form keys added as the user reveals them.
# A preference the user states is a declaration and belongs here, never in memory.md.
conventions:
  tag_scheme: "app:<git-sha>, moved tags for prod/staging only"
  labels: [org.opencontainers.image.source, org.opencontainers.image.revision]
tooling:
  compose_over_run: true
  devcontainers: false
platform:
  host_os: fedora        # SELinux — bind mounts need :z / :Z
  multi_arch: [linux/amd64, linux/arm64]
cadence:
  prune: weekly
  base_rebuild: monthly
```

If you find a preference recorded in `memory.md`, move it here and note the move.

## memory.md

Write only the sections you have content for — a heading with nothing under it is noise, and it inflates the line count that decides a split. Never copy these hints into the user's file. `## Boxes` is the one section that is never dropped when `memory.md` is rewritten: deleting a line there orphans a file forever. This is what a populated file looks like:

```markdown
# Docker Memory

## Status
status: ongoing
last: 2026-07-26

## Boxes
- Volumes and their backups (19) → `volumes.md`; read before any backup, restore or `down -v`
- Deploys and rollback digests (2026) → `deploys/2026.md`; read before any deploy or rollback
- Checkout OOM runbook → `artifacts/runbook-checkout-oom.md`; read the moment checkout restart-loops
- Hardened API Dockerfile → `artifacts/dockerfile-api.md`; read before changing the API image
- daemon.json for the build host → `artifacts/daemon-json-build-host.md`; read before touching the build host

## Due
| What | Every | Last run | Next due |
|------|-------|----------|----------|
| Prune sweep (images + build cache) | week | 2026-07-20 | 2026-07-27 |
| Rebuild and rescan base images | month | 2026-07-01 | 2026-08-01 |
| Volume restore drill (timed) | quarter | 2026-05-14 | 2026-08-14 |
| Host reboot drill | quarter | 2026-04-02 | 2026-07-02 |

## Environment
Runtime: OrbStack on macOS arm64, VM capped at 8 GB — container limits above ~7 GB are fiction.
Build host `build-1`: Ubuntu 24.04, Engine 27.x, overlay2, live-restore on.
Corporate VPN drops MTU to 1400; project networks created with the matching option.
Corp root CA mounted into build stages; `NODE_EXTRA_CA_CERTS` set in the Node images.
Host ports already taken on `app-1`: 80, 443, 5432 (non-Docker Postgres).

## Stacks
| Stack / service | Image | Base | Platform | Pin | Registry | Notes |
|---|---|---|---|---|---|---|
| checkout-api | ghcr.io/acme/checkout-api | python:3.12-slim | amd64 | digest | ghcr.io | gunicorn, 2 workers, -m 1g |
| web | ghcr.io/acme/web | node:22-slim | amd64, arm64 | digest | ghcr.io | multi-stage, distroless runtime |
| postgres | postgres:16 | — | amd64 | tag | docker.io | data on `pgdata` volume |

## Volumes
| Volume | Holds | Backup | Last backup | Restore tested |
|---|---|---|---|---|
| pgdata | Postgres 16 data dir | nightly `pg_dump` to S3 | 2026-07-25 | 2026-05-14, 11 min |
| uploads | user uploads | weekly tarball | 2026-07-21 | never |

## Registries
| Registry | Used for | Auth | Notes |
|---|---|---|---|
| ghcr.io | all first-party images | `keychain:ghcr-push` via credential helper | retention: PR tags deleted after 14 days |
| docker.io | base images only | anonymous | pulls throttled per IP on the CI NAT — mirrored through `registry-mirror.acme.internal` |

## Pain Points
2026-03: unbounded json-file logs filled `/var/lib/docker` on `app-1` and hung the daemon. Log caps now in daemon.json.
2026-06: `initdb.d` seed silently skipped twice — the volume already existed. Now a separate migration service.

## How They Work
Compose on two hosts, GitHub Actions for builds. Wants the flag and the file, not the theory. Will not run anything destructive without seeing what it deletes.

---
*Updated: 2026-07-26*
```

Rules that keep this readable next month:

- **`## Boxes`**: one line per file that exists — `<what> (<volume>) → <file>; read when <condition>`. Written in the same turn the file is created. Never delete a line without deleting the file it points to. A box with no index line does not exist.
- **`## Due`**: check it against today's date at the start of a session and state any overdue item in one line — a statement, not a question. Every recurring thing this skill schedules belongs here, and the cadences come from `cadence` in `config.yaml` when the user has declared them.
- **`## Environment`**: facts about the machines and the network that changed a decision, one line each. This is the section that stops the same MTU, VM-ceiling or CA problem from being rediscovered every few months. Anything about a *host as a machine* (provider, specs, cost, access) belongs in the shared inventory instead; what stays here is Docker-shaped.
- **`## Stacks`**: `Pin` is the strictness actually in force for that image, which may differ from `pin_policy` — record the exception and why. `Platform` lists what is actually built, not what was intended.
- **`## Volumes`**: `Restore tested` is a date and a measured duration, or the word `never`. A backup that has never been restored is a hypothesis; writing `never` is what makes it visible.
- **`## Registries`**: auth is a pointer, never a token. Record the retention policy: PR and branch tags are the disk leak that nobody owns.
- These headings are exactly the ones `stacks.md` and `volumes.md` get when their sections outgrow this file, so each split stays a copy-paste.

| Status | Meaning |
|-------|---------|
| `ongoing` | Still learning their setup |
| `complete` | Know their hosts, stacks and workflow well |

## Shared servers inventory

Lives at `~/Clawic/data/servers/servers.md` and is shared with every other infrastructure skill — the user may not have any of them installed, so the format travels with this skill.

```markdown
# Servers

| Name | Provider | Account / Project | Region | Type | Role | Monthly | Access reference |
|------|----------|-------------------|--------|------|------|---------|------------------|
| app-1 | hetzner | acme | fsn1 | CPX31 | docker host, compose prod | 15 EUR | file:~/.ssh/id_ed25519 |
| build-1 | aws | 111122223333 | eu-west-1 | c7g.large | CI build host | 55 USD | profile:build |
```

- **Identity is `Name` + `Provider`.** Read the file before adding. If that pair is already there, update the row in place — it is yours. Never touch a row whose `Provider` you did not write.
- **Retirement is part of the inventory.** When a host is decommissioned, delete its row and note the date in `## Environment` of `memory.md`. An inventory that only grows stops being an inventory.
- **Amounts carry their currency in the value** (`15 EUR`), because rows from other providers are in other currencies and someone will add the column up. An estimate carries the date it was estimated.
- **`Role` is what the machine does, and for this skill it says `docker host`** plus what it runs — the field that lets "which box is this container on" be answered without SSH.
- **Scale cut**: one row per host while there are ≤15. Past that, one file per host at `~/Clawic/data/servers/<name>.md` with the same fields, and `servers.md` becomes the index (`Name | Provider | Role | → file`). If you arrive and the folder already looks like that, follow it — do not start a parallel `servers.md`.
- **Foreign columns win.** If `servers.md` already exists with a different column set, match its columns and add anything missing as a trailing note. Never rewrite its header.
- Access reference is a pointer only. Never a key, token, or password.
- If a host belongs to a client or a tracked project, the client goes in `~/Clawic/data/contacts/contacts.md` and the project in `~/Clawic/data/projects/<project>.md`, each referenced here by name only. Never duplicate the person or the project inside a Docker file.

## artifacts/

One file per thing, at `~/Clawic/data/docker/artifacts/<kebab-name>.md`, created the first time it exists. Canonical types here: **a Dockerfile or compose file that finally worked**, **a `daemon.json`**, **a runbook for a failure that recurred**, **a base-image or hardening decision**. Every artifact opens with when to read it, and gets its `## Boxes` line in the same turn. Every secret inside is already a pointer.

```markdown
# Dockerfile — checkout-api
*Read before any change to the checkout image. Working as of 2026-07-26.*

Why it is shaped this way: multi-stage because psycopg2 needs build deps that must not ship;
non-root UID 10001 because the platform enforces runAsNonRoot; gunicorn workers pinned to 2
because os.cpu_count() reports the host's cores, not the container's limit.

...the file, with every secret replaced by its pointer...
```

```markdown
# Runbook — checkout container OOM loop
*Read when checkout restart-loops or reports exit 137. Written 2026-07-26.*

Symptom → check → fix, in order. Ends with the digest to roll back to and where it is recorded.
```

```markdown
# Decision — distroless runtime for web, slim for checkout
*Read before changing a base image. 2026-07-26.*

Decision: ...one sentence...
Rejected: alpine — musl broke the prebuilt wheels, cold build went from 90s to 7 min.
Cost: no shell in the web image; debugging is via a netshoot sidecar, which the on-call runbook covers.
Revisit when: the platform stops offering sidecars, or image size stops mattering.
```

If the user tracks this work as a project, the one-line decision summary also belongs in the shared `~/Clawic/data/projects/<project>.md`, with the full artifact staying here and referenced by name.

## deploys/

The rollback record (SKILL.md Rule 9). Append-only, one file per year, never rewritten.

```markdown
# Deploys — 2026

| Date | Service | Host | Image digest | Tag | Rollback target | Result |
|------|---------|------|--------------|-----|-----------------|--------|
| 2026-07-24 | checkout-api | app-1 | sha256:9f2c… | a41b7e | sha256:71ad… | ok |
| 2026-07-25 | web | app-1 | sha256:3d10… | 8c99f2 | sha256:9f2c… | rolled back 40 min later, CA missing in the new base |

## Drills
| Date | What was exercised | Measured | What was missing |
|------|--------------------|----------|------------------|
| 2026-07-02 | full host reboot, unattended | 3 min to all-healthy | one service still on `restart: always` |
```

- The digest is the point of the row. A row without one is a diary entry, not a rollback record.
- `Result` names what happened, including rollbacks — a deploy log that only records successes cannot answer "when did this last break".

## Split-out files

Created only by the split procedure above, never on day one. Each keeps the exact headings it had inside `memory.md`.

`stacks.md` — `## Stacks`, one `## <host>` or `## <project>` heading above it when more than one grouping exists. This is the file that answers "what do we actually build and where does it run" without reading a compose file.

`volumes.md` — `## Volumes`, plus `## Restore Log` (date, volume, method, measured duration, what was missing). The restore log is the reason this file exists: an untested backup and a tested one look identical until the day they do not.
