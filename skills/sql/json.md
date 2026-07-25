# JSON and Semi-Structured Data

The decision that matters is not how to query JSON — it is which fields should never have been JSON. A JSON column buys schema flexibility and pays with no per-key statistics, no constraints, and no cheap alteration.

Contents: Column or JSON · Engine Support · Extraction Syntax · Indexing · Containment vs Path · Updating · Constraints · Arrays and Expansion · Building JSON · Migrating Out · Traps

## Column Or JSON

Put it in a real column when any of these is true:

- You filter, join, sort, or group by it (JSON keys have no statistics, so the planner guesses and picks bad plans)
- It must be `NOT NULL`, unique, checked, or referenced by a foreign key
- It exists on every row
- It has a stable type and unit

Keep it in JSON when the shape is genuinely per-row and open-ended: third-party webhook payloads kept verbatim, user-defined custom fields, feature-flag blobs, a request snapshot for debugging, sparse attributes across thousands of product categories.

The productive middle: store the raw document AND promote the three or four fields you query into generated or plain columns. You keep fidelity and get indexes on what matters.

```sql
-- PostgreSQL >=12: promote a hot key to a real, indexable column
ALTER TABLE events ADD COLUMN user_id BIGINT
    GENERATED ALWAYS AS ((payload->>'user_id')::bigint) STORED;
CREATE INDEX idx_events_user ON events(user_id);
```

## Engine Support

| Engine | Type | Notes |
|---|---|---|
| PostgreSQL | `JSONB` (binary, indexed) and `JSON` (text) | Use `JSONB` unless you must preserve key order and whitespace exactly; `JSONB` deduplicates keys and reorders them |
| MySQL >=5.7 | `JSON` (binary internally) | No direct index on the column; index a generated column instead |
| MariaDB | `JSON` is an alias for `LONGTEXT` with a CHECK | Not the same engine-level type as MySQL; expect differences |
| SQLite >=3.38 | JSON functions built in (`json1` earlier as an extension) | Stored as `TEXT`; `JSONB` format added in 3.45 |
| SQL Server | `NVARCHAR` + `JSON_VALUE`/`OPENJSON` | Index a persisted computed column |

## Extraction Syntax

```sql
-- PostgreSQL: -> returns JSON, ->> returns text (the distinction that causes most bugs)
SELECT payload->'user'->>'name'   AS name,        -- text
       payload->'items'->0->>'sku' AS first_sku,  -- array index is 0-based
       payload #>> '{user,address,city}' AS city  -- path form
FROM events;

-- Casting is explicit and required for comparison
WHERE (payload->>'amount')::numeric > 100

-- MySQL
SELECT JSON_UNQUOTE(JSON_EXTRACT(payload, '$.user.name')) AS name,
       payload->>'$.user.name' AS same_thing        -- ->> is the unquoting shorthand
FROM events;

-- SQLite
SELECT json_extract(payload, '$.user.name') FROM events;

-- SQL Server
SELECT JSON_VALUE(payload, '$.user.name') FROM events;   -- scalars
SELECT JSON_QUERY(payload, '$.items')     FROM events;   -- objects/arrays
```

`->` vs `->>` is the number-one JSON bug: `payload->'amount' = '100'` compares a JSON number against a JSON string and is false. Use `->>` and cast, or compare JSON to JSON (`payload->'amount' = '100'::jsonb`).

## Indexing

```sql
-- PostgreSQL: GIN over the whole document, containment queries only
CREATE INDEX idx_events_payload ON events USING GIN (payload jsonb_path_ops);
SELECT * FROM events WHERE payload @> '{"status":"paid"}';

-- PostgreSQL: B-tree on one extracted key — supports equality, ranges, and sorting
CREATE INDEX idx_events_status ON events ((payload->>'status'));
SELECT * FROM events WHERE payload->>'status' = 'paid';

-- MySQL: index a generated column
ALTER TABLE events ADD COLUMN status VARCHAR(32)
    GENERATED ALWAYS AS (payload->>'$.status') STORED;
CREATE INDEX idx_events_status ON events(status);

-- SQLite: expression index
CREATE INDEX idx_events_status ON events(json_extract(payload, '$.status'));
```

- `jsonb_path_ops` is smaller and faster than the default GIN opclass but supports only containment (`@>`) — not key-existence (`?`) or path queries. Choose by the operator you actually use.
- A GIN index does not help `payload->>'k' = 'v'`, and a B-tree expression index does not help `@>`. They are different access paths; adding the wrong one changes nothing.
- The expression in the index must match the query text exactly, including the cast.
- No engine gives per-key statistics inside a JSON document: the planner uses a fixed guess for JSON predicates, so estimates are frequently far off and joins downstream get the wrong algorithm.

## Containment vs Path Queries

```sql
-- Containment: "does the document include this subtree" — GIN-indexable, no wildcards
WHERE payload @> '{"user":{"role":"admin"}}'

-- Key existence
WHERE payload ? 'discount_code'          -- top-level key
WHERE payload ?| array['a','b']          -- any of
WHERE payload ?& array['a','b']          -- all of

-- JSONPath (PostgreSQL >=12): filters and comparisons inside the document
WHERE payload @? '$.items[*] ? (@.price > 100)'
```

Containment matches nested structure, so `@> '{"a":1}'` matches `{"a":1,"b":2}` but `@> '{"a":[1]}'` requires the array to contain 1. Array containment ignores order and duplicates — a frequent source of surprise when comparing lists.

## Updating JSON

```sql
-- PostgreSQL: merge at the top level, or set a path
UPDATE users SET prefs = prefs || '{"theme":"dark"}'::jsonb WHERE id = 1;
UPDATE users SET prefs = jsonb_set(prefs, '{notify,email}', 'true', true) WHERE id = 1;
UPDATE users SET prefs = prefs #- '{legacy_flag}' WHERE id = 1;   -- delete a path

-- MySQL
UPDATE users SET prefs = JSON_SET(prefs, '$.theme', 'dark') WHERE id = 1;
UPDATE users SET prefs = JSON_REMOVE(prefs, '$.legacy_flag') WHERE id = 1;
```

- `||` merges only the top level: `{"a":{"x":1}} || {"a":{"y":2}}` yields `{"a":{"y":2}}`, dropping `x` with no error. Nested merges need `jsonb_set` per path or a recursive helper.
- `jsonb_set` returns NULL if the document is NULL — a single NULL column wipes the update. Guard with `COALESCE(prefs, '{}'::jsonb)`.
- Every JSON update rewrites the entire document: a 200 KB blob updated per request writes 200 KB per request and inflates WAL, replication traffic, and bloat. High-churn keys belong in real columns.
- Concurrent read-modify-write on the same document loses updates exactly like any other read-modify-write.

## Constraints on JSON

```sql
-- Require a key and constrain its value
ALTER TABLE events ADD CONSTRAINT events_type_valid
    CHECK (payload ? 'type' AND payload->>'type' IN ('click','view','purchase'));

-- Require it to be an object, not an array or scalar
ALTER TABLE events ADD CONSTRAINT events_payload_object
    CHECK (jsonb_typeof(payload) = 'object');
```

CHECK constraints are the only validation a JSON column gets — there is no schema. Validate the handful of fields your code depends on, and treat everything else as untrusted at read time. If you find yourself writing more than three or four such constraints, the fields belong in columns.

## Arrays and Expansion

```sql
-- PostgreSQL: one row per array element
SELECT e.id, item->>'sku' AS sku, (item->>'qty')::int AS qty
FROM events e, jsonb_array_elements(e.payload->'items') AS item;

-- Aggregate back
SELECT id, jsonb_agg(item) FROM ... GROUP BY id;

-- MySQL >=8.0.4
SELECT e.id, j.sku FROM events e,
JSON_TABLE(e.payload, '$.items[*]' COLUMNS (sku VARCHAR(64) PATH '$.sku')) j;

-- SQLite
SELECT e.id, j.value->>'sku' FROM events e, json_each(e.payload, '$.items') j;
```

`jsonb_array_elements` on a NULL or non-array value errors or returns nothing depending on the variant — use `jsonb_array_elements(COALESCE(payload->'items','[]'::jsonb))`. In a `LEFT JOIN LATERAL`, an empty array drops the parent row unless the lateral is left-joined explicitly.

## Building JSON in SQL

```sql
-- PostgreSQL: assemble a nested result in one round trip
SELECT jsonb_build_object(
         'id', o.id,
         'total', o.total,
         'items', COALESCE(jsonb_agg(jsonb_build_object('sku', i.sku, 'qty', i.qty))
                           FILTER (WHERE i.id IS NOT NULL), '[]'::jsonb)
       )
FROM orders o LEFT JOIN order_items i ON i.order_id = o.id
GROUP BY o.id;
```

The `FILTER (WHERE i.id IS NOT NULL)` is required: without it, an order with no items produces `[null]` instead of `[]`. This pattern replaces N+1 round trips for nested API responses — but it moves serialization CPU onto the database, so use it for read-heavy endpoints, not for everything.

## Migrating Out Of JSON

When a key graduates to a column:

1. Add the nullable column.
2. Backfill in batches from the JSON (`batch_size` rows per commit).
3. Dual-write both places while old code is live.
4. Switch reads to the column; verify nothing reads the JSON key.
5. Drop the key from the document in batches, or leave it as an inert copy if space is not a concern.

Steps 3 and 4 are the expand-migrate-contract shape; the only addition is that removing the key from millions of documents is itself a full rewrite of each row, so it is optional and often skipped.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `payload->'x' = 'value'` | Compares JSON to a string literal; never true | `payload->>'x' = 'value'` |
| Ordering by `payload->>'amount'` | Text ordering: `'9' > '100'` | Cast: `ORDER BY (payload->>'amount')::numeric` |
| Filtering hot paths on JSON keys | No statistics, no index unless one was built for that exact expression | Promote to a column |
| `jsonb_set` on a NULL column | Returns NULL, erasing the row's data | `COALESCE(col, '{}'::jsonb)` |
| `||` for a nested merge | Replaces the whole subtree | `jsonb_set` per path |
| Storing numbers as JSON floats | JSON numbers are doubles in many parsers; money loses cents | Store money as a string in JSON, or as a real `NUMERIC` column |
| Using JSON to avoid writing a migration | Every read now branches on missing keys, forever | Write the migration; JSON debt compounds faster than schema debt |
| Huge documents in a hot table | Every update rewrites the whole value; PostgreSQL TOASTs and de-TOASTs it | Split the large blob into its own table, joined only when needed |
