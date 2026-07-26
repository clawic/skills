# Cluster — Slots, Hash Tags, Resharding

Cluster shards the keyspace across masters. It buys capacity and horizontal write throughput, and it costs you multi-key operations, `SELECT`, and a class of assumptions your code probably makes today.

## The Model

- The keyspace is 16384 hash slots. Slot = `CRC16(key) mod 16384`. Every master owns a range of slots; replicas follow a master, not a slot.
- Clients keep a slot map and connect directly to the owner. `MOVED` means "the map is stale, refresh"; `ASK` means "this key is migrating right now, ask there for this one request only".
- Minimum sane deployment is 3 masters (plus replicas): failover requires a majority of masters to agree, and 2 masters cannot form one after a split.
- `cluster-require-full-coverage yes` (default) stops the *whole* cluster from serving when any slot is unassigned. Setting it to `no` keeps the healthy shards serving and returns errors only for the missing range — the right choice for a cache, the wrong one for a store where a silent partial keyspace is worse than an outage.
- Only database 0 exists. Any `SELECT n` in your code blocks the migration.

## What Stops Working

| Feature | In Cluster |
|---|---|
| `MGET`, `MSET`, `SUNION`, `ZUNIONSTORE`, `SMOVE`, `RENAME`, `BITOP` across keys | `CROSSSLOT` error unless all keys share a slot |
| `MULTI`/`EXEC` spanning keys | Same rule: one slot, or it fails |
| Lua scripts | All declared KEYS must hash to one slot |
| `SELECT` | Only db 0 |
| `KEYS`, `SCAN`, `DBSIZE`, `FLUSHALL` | Per node — you must iterate every master (`redis-cli --cluster call`) |
| Plain Pub/Sub | Broadcast to all nodes; use `SPUBLISH`/`SSUBSCRIBE` on Redis >=7.0 |
| Transactions with `WATCH` across entities | Only within one slot |

## Hash Tags

`{...}` in a key name makes only the braced substring hash: `app:{user:1042}:profile` and `app:{user:1042}:sessions` land in the same slot, so multi-key commands over them work.

- Tag exactly the group you access together, and nothing else.
- The failure mode is over-tagging: `app:{global}:...` on everything puts the whole dataset in one slot on one node, which cannot be rebalanced without renaming keys. A hot slot is unfixable by adding nodes.
- Distribution check: `CLUSTER COUNTKEYSINSLOT <slot>` for suspicious slots, and `redis-cli --cluster info` for key counts per node. A node holding several times the average means a tag is too coarse.
- Big-tenant skew: tagging by tenant is natural and puts your largest tenant alone on one node. Tag by a sub-entity when tenants differ by orders of magnitude.

## Operating

```bash
redis-cli --cluster create h1:6379 h2:6379 h3:6379 --cluster-replicas 1
redis-cli --cluster check h1:6379            # slot coverage, open slots, config consistency
redis-cli --cluster info h1:6379             # keys per node, slot spread
redis-cli --cluster reshard h1:6379          # move a slot range, interactive
redis-cli --cluster rebalance h1:6379        # even out slots after adding a node
redis-cli --cluster add-node hnew:6379 h1:6379
redis-cli --cluster call h1:6379 DBSIZE      # run a command on every node
CLUSTER NODES / CLUSTER SLOTS / CLUSTER SHARDS   # topology from a node's own view
```

- Adding capacity is two steps: `add-node`, then `reshard`/`rebalance`. Until slots move, a new node holds nothing.
- Resharding moves keys live, key by key (`MIGRATE`); big keys move as a single blocking transfer on both sides — another reason a 500 MB key is an operational problem.
- Removing a node: move its slots away first, then `del-node`. A node that still owns slots cannot be removed safely.
- `CLUSTER FORGET` must run on every remaining node, and there is a 60-second ban window before the node could rejoin.

## Failover

- Each master's replicas monitor it; a majority of masters must agree the master is down (`cluster-node-timeout`, default 15000 ms) before a replica is promoted.
- A master with no replica means its slots are unavailable on failure — and with `cluster-require-full-coverage yes`, that takes the cluster down.
- Replica migration moves a spare replica to a master that has none, automatically, if you have configured spares. Verify with `CLUSTER NODES` that no master is running solo.
- Manual, planned failover: `CLUSTER FAILOVER` on the replica (it coordinates with the master to avoid data loss); `FORCE`/`TAKEOVER` skip that coordination and can lose writes — reserve them for a master that is genuinely gone.
- The durability caveat is unchanged: replication is asynchronous, so a promoted replica can be missing the master's last writes.

## Client Requirements

- The client library must be cluster-aware: it discovers the topology, caches the slot map, and refreshes it on `MOVED`. A standalone client pointed at a cluster node works until the first key that lives elsewhere.
- `MOVED` reaching your application logs means the client is not refreshing, or you are using a proxy that hides the topology.
- Reads from replicas need `READONLY` on the connection *and* an application that accepts stale data.
- Connection count multiplies by node count: a pool of 50 against a 6-node cluster is 300 sockets from that process.

## Should You Shard At All?

Order the questions:

1. Does the dataset still fit one machine's RAM with the headroom of Core Rule 2? If yes, a bigger instance is cheaper than a cluster in every dimension including your time.
2. Is one core saturated? Check `INFO cpu` and `used_cpu_sys`+`used_cpu_user` against wall time, and rule out round trips and O(N) commands first — most "we need to shard" is an unpipelined client.
3. Is the failure blast radius the actual driver? Then read replicas or separate instances per workload (cache / queue / sessions) often beat one cluster, and they keep multi-key commands working.

Split by workload before splitting by slot: three purpose-built instances are simpler than one cluster and remove the noisy-neighbour problem that makes people shard in the first place.
