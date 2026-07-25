# Data Modeling — Keys, Relationships, and Normal Forms

Design order that survives contact with production: identify entities and their identity → fix cardinality → choose keys → add constraints → then, and only then, denormalize against a measured read. Reversing that order produces schemas whose bugs are unfixable without a migration.

Contents: Entity Test · Cardinality · Key Choice · Natural Keys · Normal Forms · Denormalization · Nullability · Enumerations · Junctions · Inheritance · Units and Precision · Naming · Review Checklist

## Is It An Entity Or A Column?

- It is an entity when it has an independent lifecycle, its own attributes, or something else must reference it.
- It is a column when it only ever describes exactly one parent row and is never referenced.
- The tell for a missed entity: repeated column groups (`address_line1`, `address_line2`, `address_city` appearing on three tables) or numbered columns (`phone1`, `phone2`, `phone3`). Numbered columns are a one-to-many relationship written by hand.
- The tell for an over-modeled entity: a table whose only columns are an id and a name, joined every single time, never referenced from more than one place, never edited. That is an enum (→ Enumerations).

## Cardinality: Get It Right Before Anything Else

| Relationship | Implementation | Common mistake |
|---|---|---|
| 1:1 | FK with a UNIQUE constraint, or merge into one table | Splitting for aesthetics; split only for access frequency, permissions, or genuinely optional bulk columns |
| 1:N | FK on the **many** side | Putting an array of child ids on the parent, which no FK can validate |
| M:N | Junction table with a composite PK | A comma-separated string column; every query becomes `LIKE` and nothing is enforceable |
| Optional 1:N | Nullable FK on the many side | Sentinel `0`/`-1` instead of NULL, which breaks the FK |
| Self-referencing | Nullable FK to the same table | No cycle guard; recursive reads then loop forever |

Ask "can this ever be more than one?" for every 1:1 and "will it always be exactly one?" for every required FK. Both answers change over the product's life, and the M:N junction is cheap to build up front and expensive to retrofit.

## Primary Key Choice

| Option | Pick it when | Cost |
|---|---|---|
| `BIGINT GENERATED ALWAYS AS IDENTITY` | Default for almost everything | Values are guessable and leak volume (id 4,120 tells a competitor your order count) |
| UUIDv7 / ULID | Ids generated client-side, offline, or across shards; ids exposed publicly | 16 bytes vs 8 in every index; still time-ordered so inserts stay local |
| UUIDv4 | Only when unpredictability matters more than write locality | Random insert position fragments the B-tree and inflates every index |
| Composite natural key | Pure junction tables | Every child table must carry all the columns |
| `INT` | Never for a growing table (SKILL.md rule 2) | Overflow is an outage-grade migration |

Public exposure is a separate decision from the primary key: keep `BIGINT` internally and add a unique, indexed external id (UUID or short random slug) for URLs and APIs. That gives compact joins internally and no enumeration externally.

## Natural Keys: The Honest Frontier

A natural key is defensible when the value is defined by an outside authority, immutable by that authority, and never re-issued: ISO 3166 country codes, ISO 4217 currency codes.

Values that look natural and are not: email (users change them, and case/alias normalization is a policy), phone number (reassigned), national identifiers (formats change, and they are regulated personal data), SKU (merchandising renames them), "username" (rename features exist), any code with a check digit whose standard has been revised.

When in doubt: surrogate PK plus a UNIQUE constraint on the natural key. That gives you the integrity guarantee without cascading a rename through every child row.

## Normal Forms, In The Order They Bite

- **1NF** — one value per column, no repeating groups. Violation: a comma-separated `tags` column. Cost: no index, no FK, no counting.
- **2NF** — no non-key column depends on part of a composite key. Violation: `order_items(order_id, product_id, product_name)` — `product_name` depends on `product_id` alone. Cost: renaming a product updates thousands of rows and misses some.
- **3NF** — no non-key column depends on another non-key column. Violation: storing both `zip` and `city` when `zip` determines `city`. Cost: they drift.
- **BCNF** — every determinant is a candidate key. Matters mainly for tables with overlapping candidate keys; reaching 3NF handles the vast majority of real designs.
- **4NF/5NF** — independent multi-valued facts crammed into one table (a person's languages and their skills in one row) produce a cross-product of rows. Rare, but unmistakable once seen: the row count is the product of two unrelated lists.

Design at 3NF by default. Every deliberate exception gets a comment saying which read it serves and what keeps the copies in sync.

## Denormalization: The Four Legitimate Kinds

Denormalize only after the normalized read is measured and too slow, and only with a named synchronization mechanism.

1. **Cached aggregate** (`posts.comment_count`) — synced by trigger or by the same transaction as the insert. Becomes a lock hotspot on hot rows; shard into N counter rows and SUM if it does.
2. **Copied immutable value** (`order_items.unit_price_at_purchase`) — not denormalization at all: the price at purchase time is a different fact from the current price. Always copy this one.
3. **Materialized rollup** (daily revenue table) — refreshed on a schedule, explicitly stale, queried by dashboards.
4. **Redundant FK for locality** (`comments.post_author_id` to avoid a join in a hot path) — measurable win only on very hot paths, and it must be maintained on parent change.

Anything outside these four is usually a missing index.

## Nullability

- `NOT NULL` is the default posture; nullable is the exception you justify. Every nullable column multiplies the query branches downstream (SKILL.md rule 6).
- NULL means "unknown or not applicable". It does not mean zero, empty string, or false — encoding those as NULL destroys the distinction between "no answer" and "answered zero".
- Never use a sentinel (`-1`, `'N/A'`, `1970-01-01`) to avoid NULL: it defeats aggregates, sorts wrong, and eventually collides with a real value.
- A column nullable "for now, during the migration" needs a scheduled follow-up to set `NOT NULL`, or it is permanent.
- Uniqueness over nullable columns treats NULLs as distinct in most engines — two rows with NULL both pass a UNIQUE constraint. PostgreSQL >=15 offers `UNIQUE NULLS NOT DISTINCT` when you need the opposite.

## Enumerations: Three Options

| Option | When | Watch out |
|---|---|---|
| `TEXT` + `CHECK (col IN (...))` | Default — values evolve with normal migrations | Constraint must be dropped and re-added to change the set |
| Native `ENUM` type | Fixed forever, and the storage saving matters | PostgreSQL cannot remove or reorder values; MySQL renumbers on `ALTER` |
| Lookup table + FK | The set has attributes (label, sort order, active flag) or is user-editable | An extra join on every read |

Store the machine value, not the display label — labels are localized and get edited. Transition rules between states belong in code or a trigger, never in a CHECK, which only sees the new row.

## Junction Tables

```sql
CREATE TABLE post_tags (
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  BIGINT NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);
CREATE INDEX idx_post_tags_tag ON post_tags(tag_id);   -- the reverse lookup (SKILL.md rule 4)
```

- The composite PK gives you one direction's index for free; the other direction needs its own index, and its absence is the most common junction-table performance bug.
- Add a surrogate id only when the relationship itself gains attributes (`added_at`, `added_by`, `role`) or something must reference the relationship.
- Column order in the PK is a real decision: put the side you filter on most often first (SKILL.md rule 3).

## Inheritance / Subtypes

Three shapes for "a Vehicle is a Car or a Truck":

- **Single table** — all columns in one table with a `type` discriminator. Simplest queries; subtype-specific columns must be nullable, so the database cannot enforce "trucks have `payload_kg`". Best when subtypes differ by a couple of columns.
- **Table per subtype** with a shared parent table holding the common columns and a FK. Full integrity, one join per read. Best when subtypes have substantial distinct attributes and are queried separately.
- **Table per concrete type**, no shared parent. Fastest per-type reads, but no way to reference "any vehicle" with a FK, and every cross-type query is a `UNION ALL`.

Default to single table below roughly five subtype-specific columns, table-per-subtype above it. The forcing question: does anything need a foreign key to "any subtype"? If yes, you need a shared parent table.

## Units, Precision, and Encoding of Values

- Store money as integer minor units or `NUMERIC(p, s)`, and store the currency next to it. An amount without a currency column is a bug waiting for the second market.
- Put the unit in the column name (`weight_kg`, `duration_ms`, `distance_m`) — a column named `weight` is read as pounds by someone eventually. Convert at the presentation layer only.
- Percentages: pick ratio (0-1) or percent (0-100), state it in the name (`discount_rate`, `discount_pct`), and never mix them in one schema.
- Precision for `NUMERIC(p, s)`: `s` must exceed the smallest unit you will ever aggregate. Interest and tax intermediate values typically need 4-6 decimal places even when the display shows 2.
- Booleans that will grow a third state (`is_approved` → approved/rejected/pending) start as a status enum. The retrofit costs a migration and a search for every truthiness check.

## Naming That Prevents Bugs

- One concept, one name, across the whole schema: `user_id` everywhere, never `uid` in one table and `owner` in another.
- FK column = referenced table singular + `_id` (`user_id` → `users.id`). Deviation forces every reader to check.
- Booleans read as assertions (`is_active`, `has_verified_email`); timestamps end in `_at` (`deleted_at`); dates end in `_on`; durations carry the unit.
- Name constraints and indexes explicitly (`uq_users_email_live`, `fk_orders_user`), because the engine's autogenerated name is what appears in production error messages.
- Avoid reserved words (`order`, `user`, `group`, `check`, `end`) — quoting them forever is worse than choosing `orders`, `accounts`, `groups`. Casing and quoting rules differ by engine.

## Review Checklist

- Every table has a primary key, and its type follows SKILL.md rule 2.
- Every FK column is indexed (SKILL.md rule 4), and its `ON DELETE` behavior was chosen, not defaulted.
- Every UNIQUE constraint is scoped correctly: per tenant, and excluding soft-deleted rows if they exist.
- Every nullable column has a reason; every NULL means "unknown", not zero.
- No repeating groups, no comma-separated lists, no numbered columns.
- Every timestamp is zone-aware and stored as UTC; every money column has a currency.
- Nothing stores a derived value without a stated sync mechanism.
- The model answers the three or four queries the feature actually needs — write them out and check.
