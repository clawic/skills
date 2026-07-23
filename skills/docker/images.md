# Image Building Traps

## Layer Cache

- `COPY . .` before `RUN npm install` = cache invalidated on every code change
- `apt-get update` and `apt-get install` in separate RUNs = stale packages weeks later
- `--no-cache` in build wipes ALL cache, not just the current step
- One stage's cache isn't used by another stage, multi-stage rebuild from scratch

## Multi-Stage

- `--from=builder` with a typo = silently copies from the wrong stage
- `COPY --from=0` is the first stage, not a stage named "0"
- Unnamed stage + reordering stages = `--from=N` points to a different stage
- Files copied from a previous stage lose permissions, copy with `--chmod`

## Base Images

- `python:latest` today ≠ `python:latest` tomorrow, non-reproducible builds
- `alpine` without glibc = many binaries don't work, cryptic errors
- `slim` images without shell tools = debugging impossible
- A "latest" image can be a different major version, breaking changes

## COPY vs ADD

- `ADD` with a URL downloads but doesn't cache, rebuild = re-download
- `ADD` with a .tar.gz extracts automatically, a surprise if you didn't expect it
- `COPY` doesn't expand wildcards like the shell, `COPY *.json ./` may not do what you expect
- `.dockerignore` ignored in remote builds (docker build - < Dockerfile)

## ARG vs ENV

- `ARG` not available after `FROM`, each stage needs to re-declare it
- `ARG` with a default value + empty override = uses the default, not empty
- `ARG` visible in `docker history`, not for secrets
- `ENV` persists at runtime, `ARG` only at build time

## Size Traps

- `rm -rf /var/lib/apt/lists` in a separate RUN = space not reclaimed (layers)
- `npm install --production` after `npm install` = dev dependencies still in the previous layer
- `.git` copied = extra megabytes without a .dockerignore
- Multiple `RUN apt-get` = each one is a layer with its own apt cache
