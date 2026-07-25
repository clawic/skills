# Schema Patterns

Ready-made shapes for problems that recur in every application, with the trap each one ships with. This file assumes the model is already decided (keys, normal forms, cardinality) and you need the table.

Contents: Multi-tenancy · Soft Deletes · Audit Logging · Polymorphic Associations · Tags · State Machines · Permissions · Full-Text Search · Versioning · Key-Value Settings · Feature Flags · Job Queue · Idempotency and Outbox · Rate Limits · Ledgers and Balances · Attachments · Recurrence · Time-Series · Counting

## Multi-tenancy

Default: shared tables with `tenant_id`. Schema-per-tenant only when tenants need divergent schemas or independent restore, and the tenant count stays in the hundreds — migrations, backups, and catalog overhead all scale per schema.

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    UNIQUE (tenant_id, email)      -- uniqueness is per-tenant, never global
);
-- tenant_id leads every composite index (equality-first rule, SKILL.md)
CREATE INDEX idx_users_tenant ON users(tenant_id, email);
```

A forgotten `WHERE tenant_id = ?` is a cross-tenant data leak. Enforce in the database with row-level security, not code review:

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;    -- otherwise the table owner bypasses it
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::bigint)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
-- App sets per-transaction: SET LOCAL app.tenant_id = '42';
```

One noisy tenant starving the rest is a workload problem, not a schema one.

## Soft Deletes

```sql
CREATE TABLE posts (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    deleted_at TIMESTAMPTZ         -- NULL = live
);

CREATE VIEW active_posts AS SELECT * FROM posts WHERE deleted_at IS NULL;
```

The trap that ships to prod: a plain `UNIQUE(email)` still counts soft-deleted rows, so a deleted user blocks re-registration forever. Scope uniqueness to live rows:

```sql
CREATE UNIQUE INDEX idx_users_email_live ON users(email) WHERE deleted_at IS NULL;
```

- Soft delete does not cascade: FKs still point at "deleted" parents, and `ON DELETE CASCADE` never fires. Decide per child table whether to soft-delete along or allow orphan-of-deleted.
- Every query must filter, forever. Enforce with a view or a mandatory scope, because the one report that forgets is the one shown to the customer.
- Index `deleted_at` as a partial predicate on the indexes you actually use, not as its own column index — the value is NULL for almost every row.
- If the requirement is retention or audit rather than undo, an audit log plus hard delete is simpler and erasable on request.

## Audit Logging

```sql
CREATE TABLE audit_log (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name TEXT NOT NULL,
    record_id BIGINT NOT NULL,
    action TEXT NOT NULL,          -- INSERT, UPDATE, DELETE
    old_data JSONB,                -- NULL on INSERT
    new_data JSONB,                -- NULL on DELETE
    changed_by BIGINT,             -- application user, not the database role
    changed_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id, changed_at);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by)
    VALUES (TG_TABLE_NAME, COALESCE(NEW.id, OLD.id), TG_OP,
            to_jsonb(OLD), to_jsonb(NEW),
            nullif(current_setting('app.user_id', true), '')::bigint);
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

- With a pooled connection, the database role is the same for every request — the application user must be propagated in a session variable and read by the trigger, or every audit row is useless.
- Audit tables outgrow their source tables (every UPDATE writes a row) — partition by month and drop old partitions (→ Time-Series), and never FK `changed_by` to `users` if users can be hard-deleted.
- Grant the application `INSERT` only on the audit table. An audit log the app can edit proves nothing.
- Storing full row snapshots is simple and large; storing only changed keys is compact and requires a diff at write time. Snapshots win until the table is wide.

## Polymorphic Associations

```sql
-- Type + id columns: flexible, but the database cannot enforce that
-- commentable_id points at a real row — orphans accumulate unnoticed
CREATE TABLE comments (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    body TEXT NOT NULL,
    commentable_type TEXT NOT NULL,   -- 'Post', 'Photo'
    commentable_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_comments_poly ON comments(commentable_type, commentable_id);

-- One nullable FK per target: real referential integrity; CHECK enforces exactly one
CREATE TABLE comments (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    body TEXT NOT NULL,
    post_id BIGINT REFERENCES posts(id),
    photo_id BIGINT REFERENCES photos(id),
    CHECK ((post_id IS NOT NULL)::int + (photo_id IS NOT NULL)::int = 1)
);
```

Pick separate FKs when the target set is small and stable (2-4 types); pick type+id only when types are open-ended — and accept you now own orphan cleanup, which means a scheduled data-quality query.

## Tags/Labels

```sql
-- Junction table: portable, FK integrity, tags are first-class rows
CREATE TABLE tags (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);
CREATE TABLE post_tags (
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);
CREATE INDEX idx_post_tags_tag ON post_tags(tag_id);   -- reverse lookup (SKILL.md rule 4)

-- Posts having ALL listed tags: COUNT must equal the list length
SELECT p.* FROM posts p
JOIN post_tags pt ON pt.post_id = p.id
JOIN tags t ON t.id = pt.tag_id
WHERE t.name IN ('sql', 'tutorial')
GROUP BY p.id
HAVING COUNT(DISTINCT t.name) = 2;
```

```sql
-- Array column (PostgreSQL): less machinery, GIN-indexed containment —
-- but no FK, so renaming a tag means updating every row that carries it
CREATE TABLE posts (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    tags TEXT[] DEFAULT '{}'
);
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
SELECT * FROM posts WHERE tags @> ARRAY['sql', 'tutorial'];  -- has all
```

Array when tags are free-form labels nobody manages; junction when tags have identity (rename, merge, count, permissions).

## State Machines

Native ENUM is append-only in practice: PostgreSQL `ALTER TYPE ... ADD VALUE` works, but values can never be dropped or reordered, and before PostgreSQL 12 it could not run inside a transaction (breaking migration tools). When states will evolve, prefer TEXT + CHECK:

```sql
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'pending', 'paid', 'shipped', 'delivered', 'cancelled'))
);
-- Changing states = drop + re-add the CHECK constraint, a normal migration

-- History table gives auditability and "time in state" queries
CREATE TABLE order_status_history (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    from_status TEXT,
    to_status TEXT NOT NULL,
    changed_at TIMESTAMPTZ DEFAULT NOW()
);
```

Valid transitions (draft→pending, not draft→delivered) belong in application code or a trigger — a CHECK constraint sees only the new row, not the transition. Concurrent transitions on the same row need a lock or an optimistic version check, or two workers both move it forward.

## Permissions (RBAC)

```sql
CREATE TABLE roles (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);
CREATE TABLE permissions (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT UNIQUE NOT NULL      -- 'invoice.read', 'invoice.void'
);
CREATE TABLE role_permissions (
    role_id BIGINT REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);
-- Roles are scoped: a user is an admin of one organization, not of everything
CREATE TABLE user_roles (
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT REFERENCES roles(id),
    scope_type TEXT NOT NULL,      -- 'org', 'project', 'global'
    scope_id BIGINT,
    PRIMARY KEY (user_id, role_id, scope_type, scope_id)
);
```

- Permission names are verbs on resources and are checked by exact string. Never check the role name in application code, or adding a role means editing every call site.
- Scope is the part everyone forgets: an unscoped `user_roles(user_id, role_id)` makes every admin a global admin the first time you add a second organization.
- Resolving effective permissions on every request is a join; cache it per request, and invalidate on role change.
- Row-level filtering (which invoices, not which action) is a different mechanism — RLS or an explicit predicate.
- Deny rules and role inheritance make resolution order-dependent and hard to reason about. Default to grant-only, flat roles; add hierarchy only when the role count makes it unavoidable.

## Full-Text Search

```sql
-- PostgreSQL >=12: generated column replaces the old trigger machinery
ALTER TABLE posts ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(body, '')),  'B')
    ) STORED;
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);

-- Query with ranking; websearch_to_tsquery accepts raw user input safely
SELECT *, ts_rank(search_vector, q) AS rank
FROM posts, websearch_to_tsquery('english', 'database performance') q
WHERE search_vector @@ q
ORDER BY rank DESC;
```

```sql
-- SQLite FTS5: external-content table needs sync triggers on the base table
CREATE VIRTUAL TABLE posts_fts USING fts5(title, body, content=posts, content_rowid=id);
SELECT * FROM posts_fts WHERE posts_fts MATCH 'database performance';
```

- tsvector search finds words, not substrings — "data" won't match "database". For fuzzy, substring, or typo matching use `pg_trgm`; for relevance beyond `ts_rank`, that is a search engine's job (`elasticsearch`).
- `to_tsquery` throws a syntax error on raw user input (unbalanced quotes, stray operators). `websearch_to_tsquery` (PostgreSQL >=11) parses human search syntax safely; `plainto_tsquery` ANDs all terms.
- The text search configuration (`'english'`) controls stemming and stop words. Indexing with one configuration and querying with another returns nothing at all.
- Multi-language content needs a language column and per-language indexes; one configuration stems the other languages incorrectly.

## Versioning (Keep History)

```sql
CREATE TABLE documents (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE TABLE document_versions (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    document_id BIGINT REFERENCES documents(id),
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    version INTEGER NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (document_id, version)   -- catches double-fire and race bugs
);

CREATE OR REPLACE FUNCTION save_document_version()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO document_versions (document_id, title, body, version)
    VALUES (OLD.id, OLD.title, OLD.body, OLD.version);
    NEW.version = OLD.version + 1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER documents_version
BEFORE UPDATE ON documents
FOR EACH ROW
WHEN (OLD.* IS DISTINCT FROM NEW.*)   -- no phantom versions from no-op updates
EXECUTE FUNCTION save_document_version();
```

The `version` column doubles as an optimistic lock: an update guarded by `WHERE version = :expected` fails when someone else saved first. For temporal validity ("what was the price on this date") rather than edit history, use validity ranges instead of a version counter.

## Settings/Config (Key-Value)

```sql
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value JSONB NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
INSERT INTO settings (key, value) VALUES ('user_prefs', '{"theme": "dark"}')
ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value, updated_at = NOW();

CREATE TABLE user_settings (
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    key TEXT NOT NULL,
    value JSONB NOT NULL,
    PRIMARY KEY (user_id, key)
);
```

Key-value is for genuinely open-ended settings. The moment a "setting" needs a type, a default, validation, or appears in a WHERE clause across users — promote it to a real column. EAV as the primary data model is a lot of pain for no gain.

## Feature Flags

```sql
CREATE TABLE feature_flags (
    key TEXT PRIMARY KEY,
    enabled BOOLEAN NOT NULL DEFAULT false,
    rollout_percent SMALLINT NOT NULL DEFAULT 0 CHECK (rollout_percent BETWEEN 0 AND 100),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE TABLE feature_flag_overrides (
    key TEXT REFERENCES feature_flags(key) ON DELETE CASCADE,
    scope_type TEXT NOT NULL,      -- 'user', 'tenant'
    scope_id BIGINT NOT NULL,
    enabled BOOLEAN NOT NULL,
    PRIMARY KEY (key, scope_type, scope_id)
);
```

- Percentage rollout must be deterministic per subject: hash `key || subject_id` and compare against the threshold, so a user does not flip between variants on every request.
- Resolution order is override → percentage → global default, and it must be identical everywhere the flag is read.
- Flags are read constantly and changed rarely: cache them and invalidate on write, rather than querying per request.
- Every flag needs a removal date. A flags table with entries from three years ago is a schema nobody can reason about.

## Job Queue Table

```sql
CREATE TABLE jobs (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    queue TEXT NOT NULL DEFAULT 'default',
    payload JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','running','done','failed')),
    attempts SMALLINT NOT NULL DEFAULT 0,
    max_attempts SMALLINT NOT NULL DEFAULT 5,
    run_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),     -- scheduling and backoff
    locked_at TIMESTAMPTZ,                         -- visibility timeout
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- The one index the pull query needs
CREATE INDEX idx_jobs_pull ON jobs(queue, run_at) WHERE status = 'pending';
```

- Pull with `FOR UPDATE SKIP LOCKED` so N workers take disjoint jobs with no coordinator.
- A crashed worker leaves a row `running` forever unless something reclaims it: sweep rows whose `locked_at` is older than the visibility timeout back to `pending`.
- Retries need exponential backoff written into `run_at`, and a terminal state at `max_attempts` — an infinite retry loop on a poison message saturates the queue.
- Jobs must be idempotent: at-least-once delivery is what this design provides.
- Delete or archive completed rows on a schedule. A `jobs` table that keeps every completed row becomes the largest table in the database and slows the pull query.
- This design is correct up to moderate rates; beyond that, queue churn competes with the application's own writes.

## Idempotency Keys and Outbox

```sql
-- Deduplicate externally-triggered writes: the unique constraint is the mechanism
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    endpoint TEXT NOT NULL,
    request_hash TEXT NOT NULL,        -- reject a reused key with a different body
    response JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Outbox: the external effect is committed atomically with the data change
CREATE TABLE outbox (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate TEXT NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_outbox_unpublished ON outbox(id) WHERE published_at IS NULL;
```

A database transaction cannot span an external system. Writing the event to `outbox` in the same transaction as the business change, and publishing it from a separate poller, converts "the row was saved but the webhook never fired" into "the webhook fires late" — the failure mode you can live with. Consumers must tolerate duplicates.

Expire idempotency keys on a stated window (24 hours is typical for payment APIs), or the table grows without bound.

## Rate Limits and Quotas

```sql
-- Fixed window: one row per subject per window; cheap and slightly unfair at edges
CREATE TABLE rate_limits (
    subject TEXT NOT NULL,                 -- user id, api key, ip
    window_start TIMESTAMPTZ NOT NULL,
    count INT NOT NULL DEFAULT 0,
    PRIMARY KEY (subject, window_start)
);
INSERT INTO rate_limits (subject, window_start, count)
VALUES (:subject, date_trunc('minute', NOW()), 1)
ON CONFLICT (subject, window_start) DO UPDATE SET count = rate_limits.count + 1
RETURNING count;
```

- Fixed windows allow a burst of up to 2× the limit across a window boundary. A sliding window (weighting the previous window by the elapsed fraction) fixes it at the cost of a second row read.
- The counter row is a contention point per subject under load, which is exactly the intent for one user but a problem for a global limit.
- Rate limiting in the database costs a write per request. Move it to an in-memory store once the request rate matters; keep the database version for quotas measured in days or months, where durability matters more than latency.
- Delete expired windows on a schedule, or partition by window.

## Ledgers and Balances

Never store a mutable balance as the source of truth. Store immutable entries and derive:

```sql
CREATE TABLE ledger_entries (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    account_id BIGINT NOT NULL REFERENCES accounts(id),
    transfer_id BIGINT NOT NULL,           -- groups the two sides of one movement
    amount_cents BIGINT NOT NULL,          -- signed: negative = debit
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at);

-- Double entry: every transfer's entries must sum to zero
-- (enforced by a deferred constraint trigger or by an application invariant check)
```

- Entries are append-only: corrections are new compensating entries, never updates or deletes. An editable ledger is not a ledger.
- Balance = `SUM(amount_cents)` for the account. When that scan gets expensive, add periodic snapshot rows (`balance_at(account_id, as_of, balance_cents)`) and sum only entries after the latest snapshot — the derivation stays authoritative.
- Money as integer minor units, currency stored alongside, never `FLOAT`.
- Enforce the non-negative-balance rule inside the transaction that inserts the debit, with the account row locked, or two concurrent withdrawals both pass the check.
- A scheduled reconciliation query asserting that every `transfer_id` group sums to zero catches bugs the constraints cannot.

## Attachments and Files

```sql
CREATE TABLE attachments (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    owner_type TEXT NOT NULL,
    owner_id BIGINT NOT NULL,
    storage_key TEXT NOT NULL UNIQUE,      -- path in object storage, not a URL
    filename TEXT NOT NULL,
    content_type TEXT NOT NULL,
    byte_size BIGINT NOT NULL,
    checksum TEXT NOT NULL,                -- dedupe and integrity
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- Store bytes in object storage and metadata in the database. Blobs in table rows inflate every backup, every replica, and every sequential scan.
- Store the storage key, never a full URL: hostnames, CDNs, and signing schemes change; the key does not.
- Deleting the row does not delete the object. Either delete both in a job driven by the outbox pattern, or accept orphans and run a reconciliation sweep.
- Uploads that fail halfway leave rows with no object: mark rows `pending` until the upload is confirmed, and expire stale pending rows.
- The checksum enables deduplication (same bytes, one object, many rows) and detects corruption that raises no error.

## Recurrence

```sql
CREATE TABLE recurring_events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    rrule TEXT NOT NULL,                   -- RFC 5545 rule, expanded by the application
    starts_at_local TIMESTAMP NOT NULL,
    tz TEXT NOT NULL,
    until_date DATE
);
CREATE TABLE recurring_event_exceptions (
    event_id BIGINT REFERENCES recurring_events(id) ON DELETE CASCADE,
    occurrence_date DATE NOT NULL,
    cancelled BOOLEAN NOT NULL DEFAULT false,
    override_starts_at_local TIMESTAMP,
    PRIMARY KEY (event_id, occurrence_date)
);
```

Store the rule plus exceptions, and materialize occurrences into a table only for the window you need to query (the next year, say), refreshed on rule change. Expanding every occurrence forever is unbounded; expanding none makes "what is on Tuesday" unanswerable in SQL. The local-time-plus-zone storage is required because recurrence follows the wall clock across DST.

## Time-Series Data

```sql
-- Declarative range partitioning (PostgreSQL >=10; indexes propagate to
-- partitions automatically from >=11)
CREATE TABLE metrics (
    recorded_at TIMESTAMPTZ NOT NULL,
    metric_name TEXT NOT NULL,
    value NUMERIC NOT NULL
) PARTITION BY RANGE (recorded_at);

CREATE TABLE metrics_2026_01 PARTITION OF metrics
FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

- Retention is the reason to partition: `DROP TABLE metrics_2025_07` is instant and reclaims disk; `DELETE WHERE recorded_at < ...` on the same data runs for hours and leaves bloat.
- Every query should filter on the partition key, or it scans all partitions.
- Automate partition creation (pg_partman or a scheduled migration), running at least one interval ahead of the partition it creates — the outage mode is inserts failing because next month's partition doesn't exist.
- Unique constraints must include the partition key; global uniqueness is unavailable in declarative partitioning.
- Downsample old data rather than keeping raw points forever: per-minute for a week, per-hour for a month, per-day beyond. A purpose-built store (`timescaledb`, `influxdb`, `clickhouse`) does this natively.
- Partitioning pays at scale; a table you could also just index by `recorded_at` doesn't need it yet.

## Counting (Exact vs Approximate)

```sql
-- Exact: scans (index or heap) — cost grows with table size
SELECT COUNT(*) FROM large_table;

-- Approximate (PostgreSQL, instant): planner's row estimate.
-- Accurate to autovacuum's last ANALYZE; wildly off right after a bulk load
SELECT reltuples::bigint AS estimate FROM pg_class WHERE relname = 'large_table';
```

Dashboards and "~1.2M results" UIs take the estimate; billing and invariants take the exact count. If an exact count is hot, maintain a counter row updated in the same transaction as the insert/delete — and expect that counter row to become a lock hotspot under heavy write concurrency (shard it into N rows and SUM if it does). Distinct counts do not aggregate across periods and need their own treatment.
