# Setup — SQL

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

SQL failures are quiet: the query returns rows, they are just the wrong rows. Verify shape before speed, show the read before the write, and never emit a destructive statement the user cannot preview.

## How To Load Preferences

1. Read `~/Clawic/data/sql/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `dialect: postgres`, `identifier_style: snake_case`, `table_naming: plural`, `pk_type: bigint-identity`, `destructive_guard: true`, `timezone_policy: utc`, `lock_timeout: 2s`, `batch_size: 5000`; `engine_version` and `migration_tool` unset.
3. Read `~/Clawic/data/sql/memory.md` for prior context (their schema, recurring pain points, engine version). Absence is fine; proceed without comment.
4. Infer, do not interrogate: a pasted `CREATE TABLE`, connection string, error text, or migration file names the dialect and often the version. Inference applies for the session; only a stated preference gets written.

Work from defaults immediately. Never open with questions about engine, conventions, or how cautious to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names an engine, version, migration tool, key type, naming convention, DDL lock budget, or batch size → update the matching key in `~/Clawic/data/sql/config.yaml`.
- User expresses a habit or stance (how much confirmation destructive DDL needs, whether triggers are allowed, whether every migration needs a down script) → record it under the relevant preference area (tooling, conventions, platform, safety posture, output format, work order, integrations, constraints, thresholds, cadence) in `~/Clawic/data/sql/memory.md`.
- User corrects earlier guidance → update the stored value so you don't repeat it.

If the user has said nothing, store nothing. An observation never overwrites a declared preference without asking.

## What Memory Holds

Memory holds five sections: Status, Context, Schema Seen, Pain Points, Preferences. Track the schema you have already seen (table names, key types, tenancy model), their environments (local, staging, production), recurring pain points, and how much explanation they want — but only from what they actually reveal.

Never store connection strings, passwords, hostnames, or query results containing personal data. Credentials belong in the user's own secret store, never in `~/Clawic/data/sql/`.
