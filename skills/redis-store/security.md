# Security — ACLs, Exposure, Command Policy

An unauthenticated Redis reachable from the internet is not "a misconfiguration to fix later": it is remote code execution waiting for a scanner. Treat exposure as a compromise, not a warning.

## The Exposure Attack

Redis with no password and a public bind lets anyone:

1. `CONFIG SET dir /var/lib/redis/.ssh` + `CONFIG SET dbfilename authorized_keys` + a key stored as a value + `SAVE` → their SSH key on your box.
2. `MODULE LOAD` a shared object they wrote to disk the same way → arbitrary code in the server process.
3. `FLUSHALL`, or a ransom note in key `RANSOM`, or a slow exfiltration via `SCAN`.

Mitigations that exist by default: `protected-mode yes` refuses non-loopback connections when no password and no explicit `bind` are configured. Any deployment that sets `bind 0.0.0.0` for convenience has turned it off.

If an instance was ever exposed: rotate every credential that transited it, inspect `authorized_keys` and crontabs on the host, check `MODULE LIST` and `CONFIG GET dir`, and rebuild the host rather than cleaning it.

## Baseline Hardening

- Bind to a private interface, and put the instance in a security group / firewall that admits only the application subnet. Network first — everything below is defence in depth.
- Require authentication. `requirepass` is the legacy single password; ACL users (Redis >=6) are the real mechanism.
- Disable or rename the dangerous admin surface: `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `SHUTDOWN`, `KEYS`, `MODULE`. Renaming with `rename-command` is legacy; the modern form is an ACL that never grants those categories.
- TLS in transit for anything crossing a host boundary.
- Run as a non-root user with the data directory owned by it. The attack above depends on Redis being able to write where it should not.
- No credentials in key names or values you would not want in a `MONITOR` transcript, an RDB backup, or a slowlog entry.

## ACLs

```bash
ACL SETUSER app on '>secret' '~app:*' '&app:events:*' +@read +@write +@list +@hash -@dangerous
ACL SETUSER metrics on '>secret2' allkeys +info +client|list +latency +slowlog
ACL WHOAMI
ACL GETUSER app
ACL LIST
ACL SAVE                 # persist to aclfile; without it, changes die at restart
```

- Structure: `on`/`off`, `>password`, key patterns `~pattern`, channel patterns `&pattern` (Pub/Sub is *not* covered by key patterns), command rules `+cmd`/`-cmd`/`+@category`/`-@category`.
- Start from nothing and add: `-@all +@read +@write ~app:*` beats `+@all -@dangerous`, because new commands in future versions default to denied.
- Useful categories: `@admin`, `@dangerous`, `@keyspace`, `@blocking`, `@scripting`, `@pubsub`. `ACL CAT <category>` lists exactly what a category contains on *your* version — read it rather than assuming.
- Per-service users, never one shared account: the ACL is also your audit trail and your blast-radius limit.
- Store users in an `aclfile` under configuration management. `ACL SETUSER` at runtime without `ACL SAVE` is a change that vanishes at the next restart.
- `ACL LOG` records denied attempts — a real signal that a service is doing something nobody designed.

## Scripting And Modules

- Lua runs in a sandbox with no filesystem or network access, but it runs *your* server's thread: a script is a denial-of-service vector even when it cannot escape.
- Deny `@scripting` to services that do not need it. A service that can `EVAL` can do anything its key permissions allow, in arbitrary combinations.
- `MODULE LOAD` is effectively "run this native code": it should be denied to every application user, always.

## Multi-Tenancy

- Numbered databases are not a security boundary: no ACL, no memory, no CPU separation.
- Key-prefix ACLs (`~tenant:42:*`) are a real boundary for keys, but `INFO`, `DBSIZE`, `SCAN` and Pub/Sub channels still leak shape and volume unless also restricted.
- Hard isolation between tenants means separate instances. Say that plainly rather than building prefix rules that a single `KEYS` permission undoes.

## Data At Rest

- RDB and AOF files are plaintext copies of your dataset, and so are the backups you ship elsewhere. Encrypt the volume and the backup destination.
- Redis has no per-value encryption: anything encrypted must be encrypted by the application before `SET`, which also means you cannot index, range or increment it.
- Keys are not encrypted even when values are — do not put personal data in key names.

## Audit Checklist

- Is the instance reachable only from the application subnet, verified from outside rather than assumed?
- Is `protected-mode` on, or is authentication configured because it is off?
- Does every service have its own ACL user with the smallest key pattern and command set that works?
- Are `CONFIG`, `MODULE`, `DEBUG`, `SHUTDOWN`, `FLUSH*` denied to every application user?
- Is `aclfile` (or the provider's equivalent) under version control, and does `ACL LIST` match it?
- Is TLS on for every connection that leaves a host, with certificate expiry monitored?
- Do backups live somewhere with the same access controls as the live data?
- Does `ACL LOG` get read by anyone, or is it just accumulating?
