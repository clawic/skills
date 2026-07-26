# Image Building — Traps and Cache Mastery

BuildKit is the default builder on Docker Engine >=23.0; on older engines set `DOCKER_BUILDKIT=1` to use the `--mount` features below.

**Before writing or reviewing a Dockerfile for an existing service**, read `## Stacks` in `~/Clawic/data/docker/memory.md` (or `stacks.md` if `## Boxes` points there) plus any `artifacts/dockerfile-<service>.md` it indexes: the base, the platform, the pin actually in force and the reason for any exception are recorded there. Language-specific recipes are in `languages.md`.

## Layer Cache

- `COPY . .` before `RUN npm install` = dependency reinstall on every code edit. Canonical order: manifest → install → source (→ SKILL.md rule 2).
- `COPY package.json package-lock.json ./` then install, THEN `COPY . .` — the lockfile must be in the early COPY or the cache never helps.
- `--no-cache` wipes cache for ALL steps, not the current one. To bust a single step, change its line (e.g. bump a comment or ARG above it).
- Cache keys hash file content AND metadata — a `touch` alone doesn't bust COPY cache, but a permissions change does.

## Cache Mounts (the biggest win most Dockerfiles miss)

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/root/.npm npm ci
```

Package-manager downloads survive across builds without bloating any layer — turns cold rebuilds from minutes into seconds. The mount is build-time only; nothing lands in the image.

## Multi-Stage

- `--from=builder` with a typo copies from the wrong stage with no error — no error if the path happens to exist there.
- `COPY --from=0` means the first stage by position; reorder stages and every numeric `--from` points elsewhere with no error. Always name stages.
- Stages don't share step cache with each other; a change in the builder stage doesn't invalidate the runtime stage unless copied artifacts change.
- Files copied between stages keep ownership from the source stage — `COPY --from=builder --chown=10001:10001` for the non-root runtime user.

## Base Images

- `python:latest` today ≠ tomorrow — `latest` is just a default tag name, not "most recent stable". Pinning policy: → SKILL.md rule 1.
- Alpine ships musl, not glibc: prebuilt Python wheels fall back to compiling from source (build time explodes) and some binaries segfault (exit 139).
- Ballpark for the same runtime: full image ~1 GB, slim ~150 MB, alpine ~50 MB — slim captures most of the saving without the musl tax.
- `slim`/distroless lack curl and often a shell: write healthchecks against tools that exist in the image, and debug via sidecar (→ SKILL.md Traps).

## COPY vs ADD

- `ADD` auto-extracts local `.tar.gz` — a surprise when you wanted the archive itself.
- `ADD` with a URL re-downloads every build (no cache) and can't verify integrity. Use `RUN curl -fsSL <url> -o f && echo "<sha256>  f" | sha256sum -c`.
- `COPY *.json ./` glob is evaluated by the builder, not your shell — no brace expansion, no `**` recursion.

## ARG vs ENV

- `ARG` declared before `FROM` is out of scope after it; re-declare `ARG` (no value needed) inside each stage that uses it.
- `ARG` values appear in `docker history` — never secrets (→ SKILL.md Traps for the secret-safe pattern).
- `ENV` persists into the running container; `ARG` exists only at build time. `ENV X=$X` after an `ARG X` is the idiom to carry a build value into runtime — deliberate, and visible.

## Size

- `rm -rf /var/lib/apt/lists/*` must be in the SAME RUN as the install — a later RUN can't shrink an earlier layer.
- Deleting a file in a later layer hides it but keeps the bytes; `docker history` shows each layer's true size.
- No `.dockerignore` = `.git` and dependency dirs enter the build context. Context above ~100 MB in the build output is almost always this — fix the ignore file before optimizing anything else.

**After a base image, platform or pin changes for a service** — or the first time a service is containerized — write its row in `## Stacks` of `~/Clawic/data/docker/memory.md`: service, image reference, base, platforms actually built, pin strictness in force, registry (`memory-template.md`). When the Dockerfile itself is the thing worth keeping, it goes to `artifacts/dockerfile-<service>.md` with the reasoning above the file and its `## Boxes` line in the same turn. A base-image choice re-argued every quarter is the cost of not writing down why alpine was rejected.
