# Setup — MongoDB

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

MongoDB rewards designing for the queries you will actually run and punishes relational habits applied by reflex. Be concrete: name the query shape, read the plan, give the number. Save the user from the failures that only appear at production scale — the growing array, the missing index, the single-host connection string.

## How To Load Preferences

1. Read `~/Clawic/data/mongodb/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `server_version: 7.0`, `deployment: atlas`, `driver: mongosh`, `odm: none`, `id_style: objectid`, `field_naming: camelCase`, `write_concern_default: majority`, `slow_ms: 100`, `backfill_batch: 1000`, `destructive_confirm: true`.
3. Read `~/Clawic/data/mongodb/memory.md` for prior context (their cluster shape, collections, recurring pain points). Absence is fine; proceed without comment.
4. Universal preferences (units, locale, timezone) may come from `~/Clawic/profile.yaml`. Precedence: this skill's `config.yaml` > `profile.yaml` > table default.

Work from defaults immediately. Never open with questions about their cluster, their priorities, or how proactive to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a server version, deployment target, driver language, ODM, id style, or field naming convention → update the matching key in `~/Clawic/data/mongodb/config.yaml`.
- User states a threshold (what counts as slow, batch size, pool size, lag tolerance) → update the matching key, or record it under the `Thresholds` preference area if no variable exists yet.
- User expresses a stance (whether to run mutations directly, whether secondary reads are allowed, how much explain detail to narrate) → record it under the relevant preference area (tooling, conventions, platform, risk posture, output format, integrations, restrictions, cadence) in `~/Clawic/data/mongodb/memory.md`.
- User corrects earlier guidance → update the stored value so you don't repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track the shape of their deployment (standalone, replica set, sharded, Atlas tier), the collections and query patterns that keep coming up, incidents already diagnosed, and measured baselines for the metrics in `monitoring.md` — but only from what they actually reveal.

Observations go in `memory.md`; declared preferences go in `config.yaml`. An observation never overwrites a declared preference without the user confirming.
