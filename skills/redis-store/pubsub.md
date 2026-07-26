# Pub/Sub — Fan-Out That Stores Nothing

Redis Pub/Sub is a router, not a queue: a message is delivered to whoever is connected at that instant and then forgotten. Every design question reduces to whether that is acceptable.

## What You Are Buying

- **At-most-once.** No persistence, no acknowledgement, no replay. A subscriber that is restarting, reconnecting, or too slow misses messages with no trace anywhere.
- **No backlog.** `PUBLISH` returns the number of clients that received the message — the only feedback there is. Zero means nobody was listening, and the message is gone.
- **Fan-out is free.** N subscribers on a channel each get a copy; cost is N writes to N output buffers on the single thread.
- **Ordering** is per publisher connection, not global.

Use it for: cache invalidation hints, presence and typing indicators, live dashboards, "config changed, re-read it" signals — anything where the next message repairs a missed one. Use Streams for anything else.

## Commands

```bash
SUBSCRIBE app:events:user:1042        # exact channels
PSUBSCRIBE 'app:events:*'             # pattern; matching costs per published message per pattern
PUBLISH app:events:user:1042 '{"type":"profile_updated"}'
PUBSUB CHANNELS 'app:*'               # active channels with subscribers
PUBSUB NUMSUB app:events:user:1042    # subscriber count per channel
PUBSUB NUMPAT                         # number of pattern subscriptions
```

- A connection in subscriber mode (RESP2) accepts only subscribe/unsubscribe/ping/quit commands. It cannot be shared with normal traffic — budget a dedicated connection per subscriber.
- RESP3 lifts that restriction: push messages arrive out of band and the same connection can still issue commands. Whether your client exposes this differs by library.
- `PSUBSCRIBE` cost is per pattern per published message; a hundred patterns turns every `PUBLISH` into a hundred matches on the main thread.

## The Slow-Subscriber Failure

A subscriber that reads slower than the publisher publishes accumulates in its client output buffer. The default limits for the pubsub class are `32mb 8mb 60` — hard limit 32 MB, soft limit 8 MB sustained for 60 seconds — and crossing either **disconnects the client**, with no error surfaced to the publisher (the `client-output-buffer-limit pubsub` class).

Symptoms: subscribers that "randomly" reconnect under load, gaps in event streams that correlate with traffic peaks, `client_recent_max_output_buffer` climbing in `INFO clients`.

Fixes, in order: publish less (aggregate, or publish an invalidation hint instead of a payload), read faster (do work off the socket thread), raise the limit only after the first two, or move to Streams where the backlog lives on the server and is visible.

## Cluster: Plain vs Sharded

- Plain `PUBLISH` in Cluster is broadcast to **every** node so any subscriber anywhere receives it. Correct, but the cost scales with node count and it makes Pub/Sub traffic the one thing that does not shard.
- Sharded Pub/Sub (Redis >=7.0): `SPUBLISH` / `SSUBSCRIBE` / `SUNSUBSCRIBE` route by the channel name's hash slot, so a message only travels to the shard owning that slot. Subscribers must connect to the node owning the slot — a cluster-aware client handles this, an old one does not.
- Migration rule: if channel names already contain the entity id (`app:events:user:1042`), sharded Pub/Sub is a drop-in win; if one global channel carries everything, sharding it means splitting the channel first.

## Keyspace Notifications Are Pub/Sub

Enabling `notify-keyspace-events` makes Redis publish on `__keyspace@<db>__:<key>` and `__keyevent@<db>__:<event>` channels. Everything above applies: at-most-once, nothing buffered, node-local in Cluster. The `expired` event also fires when the key is actually deleted, not at the TTL instant.

## Reliability Upgrade Path

When "we lost an event once" becomes unacceptable, the smallest change is usually not "make Pub/Sub reliable" — it is one of:

1. **Stream with a consumer group** — acks, pending list, replay from any id.
2. **Pub/Sub as a hint over a durable record** — publish "row 1042 changed", let the subscriber read the current state from the source of truth. A missed hint costs staleness until the next hint or the next poll, not data.
3. **Both** — Stream for the record, Pub/Sub for the low-latency wake-up, with polling as the floor.

Option 2 is the pattern most systems actually want: it makes message loss a latency problem instead of a correctness problem.

## Testing And Debugging

- `redis-cli SUBSCRIBE 'app:events:*'` is the fastest way to see whether anything is being published at all — before debugging the client library.
- `PUBLISH` returning 0 in production while subscribers "are connected" means they are on another node (Cluster, plain Pub/Sub misconfigured), on another database (channels are *not* namespaced by db — they are global across databases, which surprises people using `SELECT`), or already disconnected by a buffer limit.
- Channels are global across numbered databases: two tenants separated by `SELECT` share the same Pub/Sub namespace. Namespace channels by prefix instead.
