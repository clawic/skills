# redis-cli — The Forensics Toolkit

The commands that answer questions during an incident, not the basics. Everything here is safe on a live server unless flagged otherwise.

## Situational Awareness In Four Commands

```bash
redis-cli INFO                     # everything; INFO memory / clients / stats / persistence / replication for one section
redis-cli --stat                   # one line per second: keys, memory, clients, requests/s, connections
redis-cli SLOWLOG GET 10           # slowest recent commands, network time excluded
redis-cli CLIENT LIST              # who is connected, what they are running, how much they buffer
```

`--stat` is the fastest way to tell a traffic problem from a data problem: requests/s flat while latency climbs points at command cost or a stolen thread.

## Walking The Keyspace Safely

```bash
redis-cli --scan --pattern 'app:session:*'                 # SCAN under the hood, never KEYS
redis-cli --scan --pattern 'app:*' | head -100             # what the prefixes look like
redis-cli --scan --pattern 'app:tmp:*' | xargs -L 500 redis-cli UNLINK
redis-cli SCAN 0 TYPE zset COUNT 500                       # server-side type filter (>=6.0)
```

Batch at 500 keys per call: one round trip per batch, and no single `UNLINK` argument list long enough to become its own stall (→ `keys-ttl.md`).

## Finding The Problem Key

```bash
redis-cli --bigkeys        # largest key per type; a full SCAN pass, safe, takes minutes on a big keyspace
redis-cli --memkeys        # same walk ranked by MEMORY USAGE (real bytes, not element count)
redis-cli --hotkeys        # most-accessed keys; requires an LFU maxmemory-policy
redis-cli MEMORY USAGE app:user:1 SAMPLES 0   # exact bytes for one key, all elements sampled
redis-cli OBJECT ENCODING app:user:1          # listpack/intset = packed; hashtable/skiplist = promoted
redis-cli OBJECT FREQ app:user:1              # LFU counter (LFU policies only)
redis-cli OBJECT IDLETIME app:user:1          # seconds since last access (LRU policies only)
```

`--bigkeys` reports the biggest key *by element count* per type; `--memkeys` by bytes. They disagree when one key holds few but enormous values — read both before concluding.

## Latency

```bash
redis-cli --latency                 # rolling min/avg/max round trip, from where you run it
redis-cli --latency-history -i 5    # the same, in 5-second windows, to correlate with events
redis-cli --latency-dist            # spark-style distribution
redis-cli --intrinsic-latency 100   # run ON the server: what the kernel/CPU cost with no Redis work
redis-cli LATENCY LATEST            # named events (fork, expire-cycle, aof-fsync-always)
redis-cli LATENCY DOCTOR            # prose interpretation of the above
redis-cli MEMORY DOCTOR             # same idea for memory
```

`LATENCY` events require `latency-monitor-threshold` to be non-zero (default 0 = disabled).

## Bulk Operations

```bash
redis-cli --pipe < commands.txt          # mass insert, RESP-formatted, orders of magnitude faster than a loop
redis-cli --eval script.lua key1 , arg1  # note the comma separating KEYS from ARGV
redis-cli --rdb /tmp/backup.rdb          # pull a fresh snapshot over the wire, no disk access needed
redis-cli --cluster call node:6379 DBSIZE   # run a command on every cluster node
redis-cli -x SET app:blob < file.json    # last argument from stdin
redis-cli --json GET app:user:1          # JSON-formatted replies for scripting
```

`--pipe` is the loading tool: generate `SET k v` lines, feed them in, and it reports errors and replies at the end.

## Dangerous, And When They Are Justified

| Command | Cost | Use only when |
|---|---|---|
| `MONITOR` | Streams every command to you; can cut throughput by more than half | A few seconds on a staging box, or a genuinely last-resort production capture with a hard timeout |
| `KEYS pattern` | O(N) blocking scan of the keyspace | Never on a live server; `--scan` exists |
| `DEBUG SLEEP n` | Blocks the server for n seconds | Deliberately testing client timeout and failover behaviour |
| `FLUSHALL` / `FLUSHDB` | Deletes everything, `ASYNC` only moves the freeing off-thread | A scratch instance, gated by `destructive_confirm` |
| `SHUTDOWN NOSAVE` | Terminates without saving | Only way out of a script that has already written |
| `CONFIG SET` | Takes effect immediately, lost at restart without `CONFIG REWRITE` | With the rewrite, or in the provider's parameter group |
| Anything else | — | Read the complexity note in the command's documentation first |

## Connecting

```bash
redis-cli -h host -p 6379 -a "$REDIS_PASSWORD" --no-auth-warning
redis-cli -u redis://user:pass@host:6379/0
redis-cli --tls --cacert ca.crt --cert client.crt --key client.key -h host
redis-cli -c -h host          # -c follows MOVED/ASK: required against a cluster
redis-cli --user app --pass "$P" ACL WHOAMI    # confirm which ACL identity you are
```

Passing `-a` on the command line puts the password in the shell history and in `ps`. Use `REDISCLI_AUTH` in the environment, or a URI from a secret store.

## One-Liners Worth Keeping

```bash
redis-cli INFO keyspace                                   # keys= vs expires= : the TTL-less population
redis-cli INFO commandstats | sort -t= -k3 -rn | head     # slowest commands by usec_per_call
redis-cli --scan --pattern 'app:*' | wc -l                # count a prefix without KEYS
redis-cli CONFIG GET maxmemory maxmemory-policy appendonly save   # the four settings behind most incidents
redis-cli CLIENT LIST | awk '{print $2}' | cut -d= -f2 | cut -d: -f1 | sort | uniq -c | sort -rn   # connections per host
```
