# Compose Traps

## depends_on

- `depends_on: [db]` waits for the CONTAINER to start, not for the service to be ready
- `condition: service_healthy` requires a defined healthcheck; without it, it fails silently
- A circular dependency isn't an error, compose tries to resolve it and may fail randomly
- depends_on doesn't affect `docker compose run`, dependency services don't start

## Environment

- `.env` must sit next to `docker-compose.yml`, it isn't read from a subdirectory
- `${VAR}` undefined = empty string, not an error, silent bugs
- `${VAR:-default}` only applies if VAR is undefined; VAR="" uses empty, not the default
- `env_file` doesn't accept export syntax, `export VAR=x` fails

## Volumes

- Volume mount over a directory with files = the container's files disappear
- Bind mount of an empty host directory = empty container directory
- `./path` is relative to the compose file, not the cwd
- A named volume copies the container's contents the first time, not afterward

## Networks

- The default bridge has no DNS between containers, names don't resolve
- Container name ≠ service name, use the service name for DNS
- `network_mode: host` disables all of compose's networking, not just for that container
- An external network isn't created automatically, it must already exist

## Build

- `build: .` uses the Dockerfile, `build: { dockerfile: X }` for another name
- The build context is sent whole to the daemon, a large directory = slow build
- `image:` + `build:` together = build and tag with that name
- Build cache isn't shared across different compose projects by default

## Healthcheck

- A healthcheck in compose overrides the Dockerfile's
- `start_period` doesn't count toward retries, failures are ignored for the first N seconds
- `test: ["CMD", "curl", ...]`, CMD uses exec, CMD-SHELL uses the shell
- Exit code 0 = healthy, 1 = unhealthy, 2 = reserved (don't use)
