# Security — Injection, Privileges, and Personal Data

Three separate jobs that get conflated: keeping untrusted input out of the parser, keeping the application's database role from being able to do damage, and keeping personal data legible only where it must be.

Contents: Injection Surface · Beyond Placeholders · Dynamic Identifiers · Least Privilege · Role Layout · Row-Level Security · Personal Data · Erasure · Encryption · Auditing · Connection Secrets · Backup Exposure · Review Checklist

## The Full Injection Surface

Placeholders bind **values**. Everything else in a statement is code, and every one of these positions has produced real incidents:

| Position | Bindable? | Safe approach |
|---|---|---|
| `WHERE col = ?` | Yes | Placeholder |
| `IN (?, ?, ?)` | Yes, one placeholder per element | Generate the exact count, or pass an array/`= ANY($1)` where supported |
| Table or column name | No | Allowlist of literal strings in code (SKILL.md rule 1) |
| `ORDER BY <col> <dir>` | No | Map an opaque client token (`"newest"`) to a hardcoded fragment; never pass the column through |
| `LIMIT` / `OFFSET` | Usually yes | Placeholder; otherwise cast to integer and clamp to a maximum |
| `LIKE` pattern | Value is bindable, wildcards are not | Escape `%` and `_` in user input, then add your own wildcards |
| Interval/date arithmetic (`NOW() - INTERVAL '? days'`) | No, the literal is part of the syntax | Bind an integer and multiply an interval, or validate the number |
| Schema/database qualifier | No | Allowlist; never derive from a hostname or subdomain header |
| JSON path expression | Engine-dependent | Treat as an identifier: allowlist |

## Beyond Placeholders

- **Second-order injection**: input stored safely, then concatenated into a later query (a report builder reading a saved "filter" column). The stored value is as untrusted as the original request.
- **String building in stored procedures**: `EXECUTE 'SELECT ... ' || col` inside a function is injectable exactly like application code. Use `format('%I', col)` for identifiers and `%L` for literals in PostgreSQL, `QUOTENAME` in SQL Server.
- **ORM escape hatches**: `.raw()`, `.whereRaw()`, `$queryRawUnsafe`, string-built `filter` arguments. The ORM protects the paths you use through it, not around it.
- **Error messages as an oracle**: returning the database error text to the client leaks table and column names and confirms injection attempts. Log the detail, return a generic message.
- **Blind and time-based probes**: an endpoint that returns different response times or row counts is enough. Correct escaping is the defense; hiding errors is not.
- **Batch separators**: drivers that allow multiple statements per call turn a single injection into arbitrary DDL. Disable multi-statement mode unless a migration path needs it.

## Dynamic Identifiers, Done Correctly

```
ALLOWED_SORTS = {"newest": "created_at DESC", "oldest": "created_at ASC", "name": "name ASC"}
order_by = ALLOWED_SORTS.get(request.sort, "created_at DESC")   # default, never the raw input
```

The allowlist maps an opaque token to a complete, hardcoded fragment. Validating with a regex (`^[a-z_]+$`) is weaker: it still permits any existing column name, so a client can sort by `password_hash` and read it one bit at a time through ordering.

## Least Privilege for the Application Role

The default posture on most projects — the app connects as owner or superuser — means one injection is total compromise. Split the roles:

```sql
-- Owner role: owns objects, runs migrations, used only by the deploy pipeline
-- App role: no DDL, no ownership
CREATE ROLE app_rw LOGIN PASSWORD :'pw';
GRANT CONNECT ON DATABASE mydb TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_rw;

-- Future tables inherit the grant; without this, every new table breaks the app after deploy
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

-- Read-only role for analytics, dashboards, and humans
CREATE ROLE app_ro LOGIN PASSWORD :'pw2';
GRANT CONNECT ON DATABASE mydb TO app_ro;
GRANT USAGE ON SCHEMA public TO app_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_ro;
```

- On PostgreSQL, `REVOKE CREATE ON SCHEMA public FROM PUBLIC` (already the default from version 15) — otherwise any role can create objects in it.
- MySQL grants are per `user@host`: `GRANT SELECT ON mydb.* TO 'app'@'10.0.%'`. A grant to `'app'@'%'` undoes the network scoping you configured elsewhere.
- Grant `DELETE` only where the app deletes. Many applications only soft-delete, and discover the difference the day an injection tries a hard one.
- Reserve `TRUNCATE`, `DROP`, and `ALTER` for the migration role. That single split turns "attacker drops the table" into "attacker cannot".

## Row-Level Security

```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents FORCE ROW LEVEL SECURITY;   -- also applies to the table owner

CREATE POLICY tenant_read ON documents FOR SELECT
    USING (tenant_id = current_setting('app.tenant_id')::bigint);
CREATE POLICY tenant_write ON documents FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
```

- `USING` filters rows that are read or targeted; `WITH CHECK` validates rows being written. A policy with only `USING` lets a tenant insert rows attributed to another tenant.
- RLS is bypassed by superusers and, unless `FORCE` is set, by the table owner. An app connecting as the owner gets no protection at all — this is the most common way RLS is deployed, and it then does nothing at all.
- The setting must be applied per connection, inside the transaction, and cleared or overwritten on reuse. With a transaction-mode pooler, use `SET LOCAL` so it cannot leak into the next tenant's transaction.
- Policies are predicates the planner applies: a policy over a non-indexed expression makes every query on the table slow. Index the policy column (it is usually `tenant_id`, which already leads the composite indexes).
- Cross-tenant references still need checking: a FK to another tenant's row passes RLS on insert unless the policy checks the parent too.

## Personal Data: Classify Before You Design

- Tag columns holding personal data in the schema itself (a comment or naming convention) so exports, logs, and fixtures can be filtered mechanically rather than by memory.
- Do not log query parameters for statements touching those columns; slow-query logs are the classic accidental PII store.
- Test and staging environments should hold generated or masked data, never a production restore. If a production restore is unavoidable, mask as part of the restore job, not afterwards.
- Store the minimum: a birth year instead of a birth date, a hashed identifier instead of a national id, an age bucket instead of an age, when the product only needs the coarse value.
- Passwords are hashed with a slow, salted, memory-hard function by the application (bcrypt, scrypt, Argon2) — never a database `MD5`/`SHA` call, which is fast by design and often lands in the query log.

## Deletion and Erasure

- An erasure request must reach every copy: the row, the audit log, soft-deleted rows, materialized views, rollup tables, backups, replicas, and any exported extract.
- Backups are the hard part. The workable policy is a bounded retention window (state it, e.g. 30 days) plus a documented rule that restores re-apply the erasure list; per-row deletion inside historical backups is not practical.
- Prefer crypto-shredding for data that must be unrecoverable on demand: encrypt each subject's sensitive fields with a per-subject key and delete the key. Every copy becomes unreadable at once, backups included.
- Anonymization must break linkability: replacing a name while keeping a unique id, an exact timestamp, and a postcode re-identifies the person. Generalize or drop the quasi-identifiers too.
- Distinguish erasure from soft delete. `deleted_at IS NOT NULL` is still the data.

## Encryption

| Layer | Protects against | Does not protect against |
|---|---|---|
| TLS in transit | Network interception | Anything with valid credentials |
| Disk / tablespace encryption at rest | Stolen disks, discarded hardware | Any authenticated query — the database decrypts transparently |
| Column-level encryption (app-side) | A dump, a read-only leak, an over-privileged analyst | Nothing if the key sits next to the data |
| Deterministic column encryption | Same, while allowing equality lookups | Frequency analysis; equal plaintexts produce equal ciphertexts |

- Choose per column: randomized encryption for anything you never search; deterministic only for a column you must look up by exact value, accepting the leak.
- Encrypted columns cannot be range-scanned, sorted, or pattern-matched. Design the query set first — retrofitting encryption onto a column with a range filter means changing the query.
- Keys live in a key management service, not in the database, not in the repository, not in an environment variable that is echoed into logs.
- Enforce TLS on the server side (`sslmode=verify-full` on the client, `require_secure_transport` on MySQL). A client that is allowed to fall back to plaintext will, on the day the certificate expires.

## Auditing Access

- Two different needs: **data change history** (who changed which row to what — the audit table) and **access logging** (who read what, when, from where — the engine's own audit facility).
- Capture the application user, not just the database role: with a shared pooled role, every row says `app_rw`. Propagate the end-user id in a session variable (`SET LOCAL app.user_id`) and read it in the audit trigger.
- The audit table must be append-only for the app role: `GRANT INSERT` only, no `UPDATE`, no `DELETE`. An audit log the application can edit proves nothing.
- Audit tables outgrow their source tables; partition and expire them on a stated retention window.
- Log failed authentication and privilege-denied events too — successful queries alone hide the reconnaissance.

## Connection Secrets

- Credentials belong in a secret manager or the platform's secret store, injected at runtime. Never in the repository, never in a migration file, never in `~/Clawic/data/sql/`.
- Connection strings appear in process listings, crash dumps, ORM debug output, and error pages. Prefer environment-injected components over one URL string, and redact them in every log formatter.
- Rotate by supporting two valid credentials at once (add the new one, deploy, remove the old); rotation with a single credential is an outage.
- Use per-service credentials so one compromised service is one revocation, and so the audit log can attribute activity.
- Bind the database to a private network and require TLS. A managed database with a public endpoint and a strong password is one credential leak from open.

## Backups Are a Copy of Everything

- A dump has the same sensitivity as the database, with none of its access control. Encrypt at rest, restrict who can download, and log every access.
- Restore drills into a scratch environment must use masked data or an isolated network — a restore drill is the most common way production data reaches a laptop.
- Verify what a dump contains before sharing it: `pg_dump --schema-only` for schema questions, and a targeted `--table` extract instead of the whole database.

## Review Checklist

- Every user value is a placeholder; every dynamic identifier comes from an allowlist mapping to hardcoded fragments.
- The application role cannot run DDL, `TRUNCATE`, or `DROP`; a separate role owns the schema.
- `DELETE` is granted only where the application deletes.
- Multi-tenant tables enforce isolation in the database (RLS with `FORCE` and both `USING` and `WITH CHECK`), not only in code.
- Personal data columns are identified, minimized, and excluded from logs and non-production environments.
- Database errors are logged in full and returned to clients generically.
- Credentials come from a secret store, TLS is required, and the endpoint is not publicly reachable.
- The audit table is append-only for the application role and carries the end-user identity.
