# Quotas — Rate Limits, Backoff, and Batching

## The Quota Model

Two buckets deplete independently: per-project (all users of your OAuth client combined) and per-user. Per-user exhaustion under a healthy project means one identity is hot — pace that identity; per-project exhaustion means the client itself is the bottleneck.

- Services signal quota differently: Drive answers 403 with `reason: userRateLimitExceeded`; Gmail answers 429 with `rateLimitExceeded`. Both are retryable; every OTHER 403 is not (SKILL.md Rule 8).
- A second OAuth client does NOT help — quota is per-project, and both clients share it. The lever is a quota-increase request or a second project, which is a governance decision, not a workaround.

## Backoff Formula

Retry only 429 and 5xx: `sleep = min(2^attempt + rand(0..1s), 64s)`, cap at ~5 attempts, then surface the failure (documented Google backoff guidance). The jitter matters — synchronized retries from a parallel sweep re-create the spike that caused the throttle.

## Gmail's Unit Budget (worked example)

Per-user rate: 250 quota units/second — send = 100 units, get = 5, list = 5 (SKILL.md Per-API Limits).

- A mail blast saturates at 250/100 = 2.5 → roughly 2 sends/second/user sustained.
- A metadata triage sweep (`list` + `get` per message = 10 units) sustains ~25 messages/second — 10× the send rate; this is why triage is cheap and sending is the bottleneck.
- Daily ceiling regardless of pacing: 2,000 (Workspace) / 500 (consumer) sends.

## Spending Less Per Object

1. Server-side filters first: Drive `q`, Gmail `q`, Directory `query`, Reports `filters` — every object you don't receive costs zero further calls.
2. Largest page the schema allows: sweeping 5,000 Drive files at pageSize 1000 is 5 list calls; at the default 100 it is 50.
3. Cheapest format tier: Gmail `metadata` over `full` (`gmail.md`); Drive tight `fields` masks (payload and context — the quota charge is per call, not per field).
4. Incremental APIs over re-listing: Drive `changes.list`, Gmail `history.list` (`automation.md`).
5. `--page-delay` on sustained sweeps of quota-sensitive APIs — finishing 20% slower beats tripping the throttle at page 40 of 50.

## Batching

Batch endpoints bundle up to 100 inner calls per HTTP request (hard cap; Gmail guidance: 50).

- Batching saves HTTP round-trips and latency, NOT quota — every inner call bills individually.
- Inner calls fail independently: parse per-part statuses; a batch that "succeeded" can contain 30 throttled parts. Re-batch only the failed ids, with backoff.
- Gmail `batchModify` (up to 1,000 message ids per call) and `batchDelete` are true bulk endpoints, not batch wrappers — one call, one quota charge, which is why `batchModify` is the marker mechanism of choice (`gmail.md`).

## Sweep Budget Formula

Before a large job: `calls ≈ ceil(objects / pageSize) + objects × per_object_calls`. A 10,000-message triage at pageSize 500 with metadata gets: 20 lists + 10,000 gets ≈ 10,020 calls × 5 units — check the result against the per-user rate to choose `--page-delay` and batch size BEFORE starting, not after the first 429.
