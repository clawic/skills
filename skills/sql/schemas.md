# Schema Design Patterns — SQL

Contents: Multi-tenancy · Soft Deletes · Audit Logging · Polymorphic Associations · Tags · State Machines · Full-Text Search · Versioning · Key-Value Settings · Time-Series · Counting

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

A forgotten `WHERE tenant_id = ?` is a cross-tenant data leak. Enforce in the database with row-level security (PostgreSQL), not code review:

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::bigint);
-- App sets per-connection: SET app.tenant_id = '42';
-- RLS does not apply to superusers or the table owner — connect as a plain role.
```

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

Soft delete does not cascade: FKs still point at "deleted" parents, and `ON DELETE CASCADE` never fires. Decide per child table whether to soft-delete along or allow orphan-of-deleted. If the requirement is retention/audit rather than undo, an audit log (below) plus hard delete is simpler and GDPR-erasable.

## Audit Logging

```sql
CREATE TABLE audit_log (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name TEXT NOT NULL,
    record_id BIGINT NOT NULL,
    action TEXT NOT NULL,          -- INSERT, UPDATE, DELETE
    old_data JSONB,                -- NULL on INSERT
    new_data JSONB,                -- NULL on DELETE
    changed_by BIGINT,
    changed_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id, changed_at);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_data, new_data)
    VALUES (TG_TABLE_NAME, COALESCE(NEW.id, OLD.id), TG_OP,
            to_jsonb(OLD), to_jsonb(NEW));
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

Audit tables outgrow their source tables (every UPDATE writes a row) — partition by month and drop old partitions (→ Time-Series), and never FK `changed_by` to `users` if users can be hard-deleted.

## Polymorphic Associations

```sql
-- Type + id columns: flexible, but the database cannot enforce that
-- commentable_id points at a real row — orphans accumulate silently
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

Pick separate FKs when the target set is small and stable (2-4 types); pick type+id only when types are open-ended — and accept you now own orphan cleanup.

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

Native ENUM is append-only in practice: PostgreSQL `ALTER TYPE ... ADD VALUE` works, but values can never be dropped or reordered, and pre-PostgreSQL 12 it couldn't even run inside a transaction (breaking migration tools). When states will evolve, prefer TEXT + CHECK:

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

Valid transitions (draft→pending, not draft→delivered) belong in application code or a trigger — a CHECK constraint sees only the new row, not the transition.

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

tsvector search finds words, not substrings — "data" won't match "database". For fuzzy/substring/typo matching use `pg_trgm` instead; for relevance beyond `ts_rank`, that's a search engine's job.

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
- Automate partition creation (pg_partman or a cron migration) — the outage mode is inserts failing because next month's partition doesn't exist.
- Partitioning pays at scale; a table you could also just index by `recorded_at` doesn't need it yet.

## Counting (Exact vs Approximate)

```sql
-- Exact: scans (index or heap) — cost grows with table size
SELECT COUNT(*) FROM large_table;

-- Approximate (PostgreSQL, instant): planner's row estimate.
-- Accurate to autovacuum's last ANALYZE; wildly off right after a bulk load
SELECT reltuples::bigint AS estimate FROM pg_class WHERE relname = 'large_table';
```

Dashboards and "~1.2M results" UIs take the estimate; billing and invariants take the exact count. If an exact count is hot, maintain a counter row updated in the same transaction as the insert/delete — and expect that counter row to become a lock hotspot under heavy write concurrency (shard it into N rows and SUM if it does).
