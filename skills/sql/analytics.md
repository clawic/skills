# Analytical Queries — Cohorts, Funnels, and Rollups

Reporting SQL fails differently from transactional SQL: it is usually correct-looking and wrong. The recurring causes are a missing date spine, a fan-out that double-counts, and a metric whose definition drifted between two queries.

Contents: Definition First · Date Spine · Cohort Retention · Funnels · Sessionization · Running and Period Comparisons · Distinct Counts · Rollups · Materialized Views · Incremental Refresh · Star Schema · Sampling and Estimates · Traps

## Define the Metric Before Writing SQL

Write these down before the first `SELECT`; disagreements between two dashboards are almost always here, not in the code.

- **Grain**: one row per what? User-day, order, session, user-month.
- **Population**: who is included and excluded (test accounts, internal users, cancelled orders, soft-deleted rows).
- **Timestamp**: which of `created_at`, `paid_at`, `shipped_at`, `updated_at` defines the period — and in which timezone.
- **Deduplication**: is a repeat action within the window one event or many?
- **Late data**: events that arrive after the period closed — does the number get restated or frozen?

A metric without these five written down cannot be reproduced, and every "the numbers don't match" investigation resolves to one of them.

## The Date Spine (the bug that never raises an error)

Grouping by a timestamp column only produces rows for periods that had data. Days with zero activity vanish, so a chart draws a line straight through the outage and an average divides by the wrong denominator.

```sql
-- PostgreSQL
SELECT d.day, COALESCE(COUNT(o.id), 0) AS orders
FROM generate_series(DATE '2026-01-01', DATE '2026-01-31', INTERVAL '1 day') AS d(day)
LEFT JOIN orders o ON o.created_at >= d.day AND o.created_at < d.day + INTERVAL '1 day'
GROUP BY d.day ORDER BY d.day;

-- Portable: recursive CTE spine (MySQL >=8.0, SQLite, SQL Server)
WITH RECURSIVE days(day) AS (
    SELECT DATE '2026-01-01'
    UNION ALL SELECT day + 1 FROM days WHERE day < DATE '2026-01-31'
)
SELECT days.day, COUNT(o.id) AS orders
FROM days LEFT JOIN orders o ON o.created_at >= days.day AND o.created_at < days.day + 1
GROUP BY days.day;
```

Join the spine on a half-open range, never on a cast (`DATE(o.created_at) = d.day` is not sargable). A permanent `calendar` table with columns for week, month, quarter, fiscal period, and holiday flags removes this boilerplate everywhere and makes fiscal calendars possible at all.

## Cohort Retention

```sql
WITH first_seen AS (
    SELECT user_id, DATE_TRUNC('month', MIN(created_at)) AS cohort_month
    FROM orders GROUP BY user_id
),
activity AS (
    SELECT DISTINCT o.user_id, DATE_TRUNC('month', o.created_at) AS active_month
    FROM orders o
)
SELECT f.cohort_month,
       -- whole months between cohort and activity; integer month offset, not day/30
       (DATE_PART('year',  a.active_month) - DATE_PART('year',  f.cohort_month)) * 12
     + (DATE_PART('month', a.active_month) - DATE_PART('month', f.cohort_month)) AS month_offset,
       COUNT(DISTINCT a.user_id) AS active_users,
       MAX(COUNT(DISTINCT a.user_id)) OVER (PARTITION BY f.cohort_month) AS cohort_size,
       ROUND(100.0 * COUNT(DISTINCT a.user_id)
             / MAX(COUNT(DISTINCT a.user_id)) OVER (PARTITION BY f.cohort_month), 1) AS retention_pct
FROM first_seen f
JOIN activity a ON a.user_id = f.user_id
GROUP BY f.cohort_month, month_offset
ORDER BY f.cohort_month, month_offset;
```

- The denominator is the cohort's size at offset 0, taken from the same query. Computing it separately guarantees the two drift.
- Truncate the table at the last **complete** period. A cohort whose month is still running always looks like a collapse, and someone will act on it.
- Rolling retention ("active on day 30 or later") and bounded retention ("active on day 30 exactly") give very different curves. Say which one the chart shows.

## Funnels

```sql
-- Ordered funnel: each step must occur after the previous one for the same user
WITH steps AS (
    SELECT user_id,
           MIN(CASE WHEN event = 'view'     THEN occurred_at END) AS t_view,
           MIN(CASE WHEN event = 'add_cart' THEN occurred_at END) AS t_cart,
           MIN(CASE WHEN event = 'purchase' THEN occurred_at END) AS t_buy
    FROM events
    WHERE occurred_at >= :from AND occurred_at < :to
    GROUP BY user_id
)
SELECT COUNT(*) FILTER (WHERE t_view IS NOT NULL) AS viewed,
       COUNT(*) FILTER (WHERE t_cart > t_view)    AS carted,
       COUNT(*) FILTER (WHERE t_buy  > t_cart)    AS purchased
FROM steps;
```

- Unordered funnels (counting anyone who did each step, in any order) inflate every stage and can report more purchases than views. Enforce the time ordering unless the product genuinely allows entry mid-funnel.
- Decide the attribution window: steps completed a month apart are usually not one funnel. Add `AND t_buy < t_view + INTERVAL '7 days'`.
- Users who entered before the window started are cut off at the left edge; either exclude them or widen the lookback for step 1 only.

## Sessionization

```sql
-- 30-minute inactivity gap defines a session boundary
WITH marked AS (
    SELECT user_id, occurred_at,
           CASE WHEN occurred_at - LAG(occurred_at) OVER (PARTITION BY user_id ORDER BY occurred_at)
                     > INTERVAL '30 minutes'
                  OR LAG(occurred_at) OVER (PARTITION BY user_id ORDER BY occurred_at) IS NULL
                THEN 1 ELSE 0 END AS is_new_session
    FROM events
)
SELECT user_id, occurred_at,
       SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY occurred_at
                                 ROWS UNBOUNDED PRECEDING) AS session_number
FROM marked;
```

The 30-minute gap is a convention from web analytics, not a law — it is wrong for products used in long background sessions. State the threshold with the metric, and keep it identical across every query that reports sessions.

## Running Totals and Period Comparisons

```sql
SELECT day, revenue,
       SUM(revenue) OVER (ORDER BY day ROWS UNBOUNDED PRECEDING) AS running_total,
       AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7,
       revenue - LAG(revenue, 7) OVER (ORDER BY day) AS wow_change
FROM daily_revenue;
```

- The default window frame is `RANGE`, which treats tied ordering values as one peer group — every row of the same day gets the same "running" total. Use `ROWS` for a true row-by-row accumulation.
- `LAG(revenue, 7)` compares to seven **rows** back, which equals seven days only if the spine has no gaps. This is why the date spine comes first.
- A moving average over a partial trailing window (the first 6 days) is computed over fewer rows and is not comparable; null it out or start the chart later.

## Counting Distinct Things

- `COUNT(DISTINCT x)` cannot be summed across partitions: daily distinct users do not add up to monthly distinct users. Every period needs its own pass, which is why "monthly active" cannot be derived from a daily rollup table without storing the identities.
- To make it additive, store a sketch instead of a count: PostgreSQL `hll` or `datasketches` extensions, ClickHouse/BigQuery natives. Sketches merge across periods with a stated error bound (HyperLogLog is typically within a couple of percent at default precision).
- Cheaper alternative for modest cardinality: store the distinct id list per period as an array or bitmap, then union at read time.
- `COUNT(DISTINCT)` over a large table is a sort or hash of all values — usually the slowest node in a report.

## Rollup Tables

The default answer for a dashboard that reads the same aggregation repeatedly.

```sql
CREATE TABLE daily_revenue (
    day DATE NOT NULL,
    tenant_id BIGINT NOT NULL,
    orders INT NOT NULL,
    revenue NUMERIC(14,2) NOT NULL,
    computed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (day, tenant_id)
);

-- Idempotent refresh of one day: safe to re-run, safe to backfill
INSERT INTO daily_revenue (day, tenant_id, orders, revenue)
SELECT DATE(created_at), tenant_id, COUNT(*), SUM(total)
FROM orders
WHERE created_at >= :day AND created_at < :day + INTERVAL '1 day'
GROUP BY 1, 2
ON CONFLICT (day, tenant_id) DO UPDATE
SET orders = EXCLUDED.orders, revenue = EXCLUDED.revenue, computed_at = NOW();
```

- Make every refresh idempotent and re-runnable for an arbitrary date. A rollup you cannot recompute for last Tuesday is unfixable when last Tuesday was wrong.
- Recompute a trailing window (default 3-7 days; a stated refresh Cadence overrides it), not only yesterday: late-arriving events change closed periods.
- Store `computed_at` so a stale dashboard is visibly stale rather than quietly wrong.
- Keep raw data as long as retention allows; a rollup is a cache, and every rollup eventually needs a new dimension.

## Materialized Views

```sql
CREATE MATERIALIZED VIEW mv_daily_revenue AS SELECT ...;
CREATE UNIQUE INDEX ON mv_daily_revenue (day, tenant_id);   -- required for CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
```

- Plain `REFRESH` takes an exclusive lock for the full rebuild — the view is unreadable while it runs. `CONCURRENTLY` avoids that but requires a unique index and is slower.
- PostgreSQL materialized views never refresh themselves; something must schedule it, and nothing warns you when that job dies. `computed_at` in a rollup table beats a materialized view precisely because staleness is visible.
- Choose a materialized view when the query is complex and full recomputation is cheap; choose a rollup table when the data is append-heavy and incremental refresh is the point.
- MySQL has no materialized views — use a rollup table. SQL Server indexed views refresh synchronously on write, which shifts the cost onto every insert.

## Star Schema, When Reporting Grows Up

- Facts are events with measures and foreign keys (one row per order line); dimensions are the descriptive entities (customer, product, date, store).
- The dimension that repays itself immediately is the date dimension: week/month/quarter, fiscal period, holiday, day-of-week, all pre-computed and joinable.
- Slowly changing dimensions: type 1 overwrites (you lose history); type 2 adds a row per version with validity dates (a fact joins to the version current at the fact's timestamp). Choose per attribute — a customer's name is type 1, their pricing tier is type 2.
- Keep facts at the finest grain you will ever need; aggregation up is trivial, disaggregation is impossible.
- On a transactional database, a star schema is often overkill — rollup tables get you most of the way. Move to a warehouse when reporting load competes with production traffic or when the transformation layer needs testing and lineage (`dbt`).

## Sampling and Estimates

- Exact `COUNT(*)` on a huge table scans it; the planner's estimate is instant and accurate to the last `ANALYZE`.
- `TABLESAMPLE SYSTEM (1)` samples pages (fast, clustered bias); `BERNOULLI (1)` samples rows uniformly and scans more.
- A sampled metric needs its error stated. A 1% sample of a 100k-row table gives roughly 1,000 rows; a proportion measured on 1,000 rows has a margin of error of about ±3 percentage points at 95% confidence (≈ 1/√n). Do not report a 0.5% change from that sample.
- Billing, invoices, and compliance figures are never sampled or estimated.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `GROUP BY` a timestamp column with no spine | Empty periods disappear; averages divide by the wrong count | Join a generated date spine or a calendar table |
| Summing daily distinct counts into a monthly figure | Distinct counts are not additive | Recompute per period, or store a sketch |
| Two dashboards, two copies of the metric SQL | Definitions drift; nobody can say which is right | One rollup table or one view as the single definition |
| Joining fact to fact | Two 1:N joins multiply rows before aggregation | Aggregate each fact to a common grain, then join |
| Reporting the current, incomplete period next to complete ones | Always looks like a crash | Exclude it, or label it explicitly as partial |
| Percent change from a tiny base | 1 → 3 is "+200%" and means nothing | Show absolute values alongside, suppress below a stated minimum base |
| Averaging an average | Unweighted means of group means ignore group sizes | Sum the numerators and denominators, then divide |
| Running analytics on the primary at business hours | Long scans evict the OLTP working set from cache | Replica, rollup, or warehouse |

Every period boundary in this file is half-open for one reason: `BETWEEN` on a timestamp silently drops the last day (→ SKILL.md Traps).
