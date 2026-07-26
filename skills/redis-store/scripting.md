# Atomicity — MULTI, Lua, Functions

Three tools, one question each: do you need commands grouped (MULTI), logic between reads and writes (Lua), or reusable server-side logic with a name (Functions)?

## MULTI/EXEC Is Not A Transaction With Rollback

```bash
MULTI
INCR app:counter
LPUSH app:log "x"
EXEC          # both run, with nothing else interleaved
```

- What it guarantees: the queued commands execute consecutively, with no other client's command in between.
- What it does not guarantee: rollback. A command that fails at runtime (wrong type, OOM) leaves the earlier commands applied and the later ones still run. Only a *syntax* error at queue time aborts the whole block (`EXECABORT`).
- You cannot branch: results are only available after `EXEC`, so "read X, decide, write Y" is impossible inside MULTI. That is what Lua is for.
- Pipelining and MULTI are different things: a pipeline batches round trips, MULTI groups execution. Most clients let you pipeline a MULTI block — you usually want both.

## WATCH: Optimistic Concurrency

```bash
WATCH app:balance:42
current = GET app:balance:42
# decide in the client
MULTI
SET app:balance:42 <new>
EXEC          # nil reply = a watched key changed; retry the whole thing
```

- `EXEC` returning nil is normal, not an error: retry with backoff, and cap the retries or a hot key becomes a livelock.
- `WATCH` is cancelled by `EXEC`, `DISCARD`, `UNWATCH`, or the connection closing — never leave a watch dangling on a pooled connection.
- Cost model: cheap when contention is rare, worse than a Lua script when it is common. Contention above roughly a few percent of attempts means switch to Lua.

## Lua Scripts

```lua
-- checked_incr.lua: KEYS[1]=counter, ARGV[1]=limit
local n = tonumber(redis.call('GET', KEYS[1]) or '0')
if n >= tonumber(ARGV[1]) then return 0 end
redis.call('INCR', KEYS[1])
return 1
```

```bash
SCRIPT LOAD "$(cat checked_incr.lua)"      # returns the sha1
EVALSHA <sha1> 1 app:limit:user:1042 100   # numkeys, then keys, then args
```

Rules that are not optional:

- **Every key the script touches goes in KEYS, and `numkeys` must be right.** Cluster routes by the declared keys; a key built inside the script from ARGV works on standalone and breaks the day you shard.
- **Scripts block the server for their whole duration.** Bound every loop. A script iterating a million-element set is a million-element stall — the same arithmetic as Core Rule 4.
- **Deterministic by construction.** Modern Redis replicates the *effects*, not the script, so `TIME` and randomness no longer corrupt replicas, but sorting a set's iteration order still makes behaviour untestable. Pass the timestamp in as an argument when the logic depends on it.
- **`EVALSHA` needs a fallback.** After a restart or `SCRIPT FLUSH` the cache is empty and you get `NOSCRIPT` — every client should catch it and re-`EVAL` (most libraries do; verify yours).
- **Errors abort the script mid-flight** with earlier writes already applied — same non-rollback rule as MULTI. Validate before writing anything.

## When A Script Runs Too Long

- After `busy-reply-threshold` (default 5000 ms; called `lua-time-limit` before Redis 7) the server starts replying `BUSY` to other clients while the script continues.
- `SCRIPT KILL` stops it **only if it has not written yet**. Once it has written, the only way out is `SHUTDOWN NOSAVE` — which loses everything since the last save.
- That asymmetry is the argument for bounded loops and for testing scripts against production-sized keys, not toy ones.

## Redis Functions (>=7.0)

```lua
#!lua name=applib
redis.register_function('claim_job', function(keys, args)
  local due = redis.call('ZRANGEBYSCORE', keys[1], '-inf', args[1], 'LIMIT', 0, 1)
  if #due == 0 then return nil end
  redis.call('ZREM', keys[1], due[1])
  return due[1]
end)
```

`FUNCTION LOAD` registers a named library that is **persisted in RDB and replicated**, so it survives restarts and exists on replicas — the operational problem `EVALSHA` has. Manage with `FUNCTION LIST`, `FUNCTION STATS`, `FUNCTION DUMP`/`RESTORE`.

Choose Functions when the logic is part of the system's contract (a claim protocol, a rate limiter) and Lua `EVALSHA` when it is an application detail shipped with the app.

## Choosing Between Them

| Need | Tool |
|---|---|
| Two writes must not be interleaved, no logic between them | `MULTI`/`EXEC` |
| A single atomic command already exists (`INCR`, `SET NX`, `LMOVE`, `ZADD GT`) | Use it — no transaction needed |
| Read, decide, write, low contention | `WATCH` + retry |
| Read, decide, write, high contention or many keys | Lua |
| Same server-side logic used by several services, must survive restart | Function |
| Anything else | Prefer the single atomic command; reach for Lua only when no command expresses it |

## Script Hygiene

- Keep scripts in files in the repository, not in string literals — they are code and need review and tests.
- Version them: a changed script gets a new sha, and old clients keep calling the old one during a rolling deploy. Make the change backward compatible or gate it.
- Return simple types: Lua tables convert to arrays and stop at the first `nil`; a table with a `nil` in the middle truncates the reply at that point and reports no error.
- Lua numbers convert to integers when returned (the fractional part is dropped) — return floats as strings.
- Watch `used_memory_scripts` and `INFO commandstats` for `eval`/`evalsha`: a client that sends the whole script body on every call is paying script-compilation cost per request.
- Managed platforms may restrict or disable scripting; check before making it load-bearing.
