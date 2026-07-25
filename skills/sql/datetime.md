# Dates, Times, and Timezones

Every timestamp bug traces to one of three questions never answered: what instant is stored, in whose calendar is it interpreted, and where is the boundary of "a day". Answer all three in the schema and most of these disappear.

Contents: Store UTC · Type Selection · The Wall-Clock Exception · Conversion · Day Boundaries · Truncation and Grouping · DST · Intervals and Age · Ranges · Business Days · Fiscal and ISO Weeks · Comparing and Indexing · Dialect Table · Traps

## Store the Instant in UTC

The default with no exceptions worth taking casually: store an absolute instant in UTC, convert at the edges, and never store a local time without also storing which zone it means.

```sql
CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW()   -- PostgreSQL: UTC instant
);
```

An offset is not a timezone. `-05:00` tells you nothing about what that clock reads next November; `America/New_York` does. Store the IANA zone name when the zone matters, never the offset.

## Type Selection

| Need | PostgreSQL | MySQL | SQLite | SQL Server |
|---|---|---|---|---|
| Absolute instant | `TIMESTAMPTZ` | `TIMESTAMP` (UTC internally, converted by session zone; range ends 2038) or `DATETIME` holding UTC | `TEXT` ISO-8601 UTC, or INTEGER epoch | `DATETIMEOFFSET` |
| Naive local wall clock | `TIMESTAMP` | `DATETIME` | `TEXT` | `DATETIME2` |
| Calendar date only | `DATE` | `DATE` | `TEXT` `YYYY-MM-DD` | `DATE` |
| Time of day only | `TIME` | `TIME` | `TEXT` | `TIME` |
| Duration | `INTERVAL` | integer seconds | integer seconds | integer seconds |

- PostgreSQL `TIMESTAMPTZ` does **not** store a zone. It converts the input to UTC on write and renders it in the session's `TimeZone` on read. The name misleads everyone once.
- MySQL `TIMESTAMP` converts on both write and read using the session timezone and tops out in 2038. `DATETIME` stores exactly what you gave it with no conversion — which is why MySQL projects usually standardize on `DATETIME` holding UTC plus an application rule.
- SQLite has no date type; text in strict `YYYY-MM-DD HH:MM:SS` sorts and compares correctly, epoch integers are compact. Pick one per database and never mix.
- Durations as `INTERVAL` are convenient but not portable and awkward to aggregate — integer seconds (or milliseconds, named in the column: `duration_ms`) travel everywhere.

## The Wall-Clock Exception

Some events are defined by local time, not by an instant. A 09:00 recurring meeting in Berlin stays at 09:00 through a DST change; the UTC instant moves.

Store both parts:

```sql
CREATE TABLE appointments (
    starts_at_local TIMESTAMP NOT NULL,           -- naive wall clock
    tz TEXT NOT NULL,                             -- IANA name, e.g. 'Europe/Berlin'
    starts_at_utc TIMESTAMPTZ                     -- resolved instant, recomputed if rules change
        GENERATED ALWAYS AS (starts_at_local AT TIME ZONE tz) STORED
);
```

The generated column gives you an indexable instant for "what is next" queries while the local pair remains the source of truth. Governments change timezone rules with months of notice — the resolved instants must be recomputable, which is exactly why the local time is stored.

Birthdays, contract dates, and holidays are `DATE`, not timestamps. Storing a birthday as a timestamp makes it shift a day for some users forever.

## Conversion

```sql
-- PostgreSQL: AT TIME ZONE flips meaning based on the input type
SELECT occurred_at AT TIME ZONE 'America/New_York';   -- timestamptz → naive local time
SELECT local_ts    AT TIME ZONE 'America/New_York';   -- naive → timestamptz (instant)

-- Session default for rendering
SET TIME ZONE 'UTC';

-- MySQL (requires the timezone tables to be loaded, or only offsets work)
SELECT CONVERT_TZ(created_at, '+00:00', 'America/New_York');

-- SQLite
SELECT datetime(occurred_at, 'localtime');            -- server's zone, not the user's

-- SQL Server
SELECT created_at AT TIME ZONE 'UTC' AT TIME ZONE 'Eastern Standard Time';
```

MySQL's `CONVERT_TZ` returns NULL when the named zone is unknown — a fresh server without `mysql_tzinfo_to_sql` loaded turns every converted timestamp into NULL. Test with a named zone, never with an offset, or the failure hides until production.

## Day Boundaries

"Today" is a range in a specific zone, and the range is what the query must express:

```sql
-- Today in the user's zone, expressed as an instant range (index-usable)
WHERE occurred_at >= (DATE '2026-03-15')::timestamp AT TIME ZONE :user_tz
  AND occurred_at <  (DATE '2026-03-16')::timestamp AT TIME ZONE :user_tz
```

- Never `WHERE DATE(occurred_at) = '2026-03-15'`: it applies a function to the column (no index) and uses the *session* zone, so the same query returns different rows for different connections.
- A daily report and a dashboard that disagree by a few rows are almost always splitting the day at different zones.
- Pick one reporting zone per tenant or per report and record it with the metric. "UTC days" is a legitimate choice; an unstated choice is not.

## Truncation and Grouping

```sql
DATE_TRUNC('month', occurred_at)                            -- PostgreSQL, also 'week','quarter'
DATE_TRUNC('day', occurred_at AT TIME ZONE 'Europe/Madrid') -- truncate in the reporting zone
DATE_FORMAT(created_at, '%Y-%m-01')                         -- MySQL month bucket
strftime('%Y-%m', occurred_at)                              -- SQLite
DATETRUNC(month, created_at)                                -- SQL Server 2022+; earlier: EOMONTH/DATEADD
```

`DATE_TRUNC('week', ...)` starts on **Monday** in PostgreSQL (ISO). MySQL's `WEEK()` defaults to Sunday and takes a mode argument controlling both the first day and how the first week of the year is counted. A "weekly" chart that disagrees between two systems is nearly always this.

Truncate in the reporting zone, then group — grouping UTC-truncated days and relabelling them locally is off by the offset for part of every day.

## Daylight Saving Time

- Some local times do not exist (the spring-forward gap) and some occur twice (the autumn repeat). Converting a naive local time to an instant is therefore ambiguous or invalid for two hours a year; engines resolve it by rule, not by asking. Store the intended instant for anything that must not shift.
- A "day" is not always 24 hours: adding `INTERVAL '1 day'` in PostgreSQL adds a calendar day (23 or 25 hours across a transition), while adding `INTERVAL '24 hours'` adds exactly 24. Pick according to whether you mean "same time tomorrow" or "one day later".
- Scheduled jobs at 02:30 local never run on the spring-forward day and run twice on the autumn day. Schedule recurring jobs in UTC.
- Durations computed by subtracting local times cross transitions incorrectly. Subtract instants.
- Zone rules change: countries abolish DST, shift offsets, and split zones. The tzdata package must be updated on database servers too, not only on application hosts.

## Intervals and Age

```sql
-- Difference as an interval, and as a number
SELECT ended_at - started_at AS duration;                       -- PostgreSQL interval
SELECT EXTRACT(EPOCH FROM (ended_at - started_at)) AS seconds;  -- PostgreSQL
SELECT TIMESTAMPDIFF(SECOND, started_at, ended_at);             -- MySQL
SELECT (julianday(ended_at) - julianday(started_at)) * 86400.0; -- SQLite
SELECT DATEDIFF(second, started_at, ended_at);                  -- SQL Server
```

- SQL Server's `DATEDIFF(year, a, b)` counts **boundary crossings**, not elapsed years: 2025-12-31 to 2026-01-01 is 1 year. MySQL's `TIMESTAMPDIFF` counts complete units. They disagree by design.
- Age in whole years is not `days / 365.25`: leap years make that wrong for some birthdays. Use the engine's age function, or compare the month-day pair explicitly.
- "Months between" is genuinely ambiguous (Jan 31 plus one month). Adding a month clamps to the last valid day in most engines, so `Jan 31 + 1 month - 1 month` does not return Jan 31. Never round-trip through month arithmetic.
- Intervals are not additive with fixed conversions: `INTERVAL '1 month'` has no fixed number of days until anchored to a date.

## Ranges and Overlap

```sql
-- Portable overlap test between [a_start, a_end) and [b_start, b_end)
WHERE a_start < b_end AND a_end > b_start

-- PostgreSQL range types handle inclusivity explicitly
WHERE tstzrange(start_at, end_at, '[)') && tstzrange(:from, :to, '[)')

-- Prevent double booking in the database rather than in code
ALTER TABLE bookings ADD CONSTRAINT no_overlap
    EXCLUDE USING gist (room_id WITH =, tstzrange(start_at, end_at, '[)') WITH &&);
```

Use half-open `[)` ranges everywhere: they tile without gaps or overlaps, so back-to-back bookings do not collide and `BETWEEN`'s inclusive upper bound never bites (SKILL.md Traps). Hand-written overlap conditions typically miss the containment case; the two-comparison form above covers all four.

## Business Days and Holidays

Business-day arithmetic in pure SQL is either wrong or unreadable. Build a calendar table once:

```sql
CREATE TABLE calendar (
    day DATE PRIMARY KEY,
    is_business_day BOOLEAN NOT NULL,
    iso_week INT NOT NULL,
    month_start DATE NOT NULL,
    fiscal_year INT NOT NULL,
    fiscal_quarter INT NOT NULL
);
-- "5 business days after 2026-03-10"
SELECT day FROM calendar WHERE is_business_day AND day > DATE '2026-03-10'
ORDER BY day OFFSET 4 LIMIT 1;
```

Holidays are jurisdictional and change yearly; a calendar table is data you maintain, not logic you derive. It also doubles as the date spine for reporting and the date dimension of a star schema.

## Fiscal Calendars and ISO Weeks

- ISO 8601: weeks start Monday, and week 1 is the week containing the first Thursday. Consequence: the first days of January can belong to week 52/53 of the previous ISO year. Always pair `EXTRACT(WEEK ...)` with `EXTRACT(ISOYEAR ...)` — using the calendar year with the ISO week misplaces early-January rows by a full year, every year.
- Some years have 53 ISO weeks. Any "week over week, 52 rows" assumption breaks in those years.
- Fiscal years rarely start in January, and retail 4-4-5 calendars are not derivable from dates at all. Put them in the calendar table.
- Quarter boundaries differ between fiscal and calendar quarters; label which one a chart shows.

## Comparing and Indexing Timestamps

- Compare instants to instants. A `TIMESTAMPTZ` compared against a naive string literal is interpreted in the session zone — the same query then behaves differently for two connections.
- Half-open ranges keep the index usable and avoid boundary loss; functions on the column disable the index entirely.
- Index the raw column and range-scan it, rather than indexing `DATE(col)` — the raw index serves both day queries and arbitrary ranges.
- For very large append-only tables ordered by time, partition by range on the timestamp and confirm every query filters on the partition key, or BRIN-index it.
- Beware `NOW()` inside a transaction: PostgreSQL's `NOW()`/`CURRENT_TIMESTAMP` is frozen at transaction start, so every row inserted in a long transaction shares one timestamp. `clock_timestamp()` gives real wall time.

## Dialect Quick Table

| Operation | PostgreSQL | MySQL | SQLite | SQL Server |
|---|---|---|---|---|
| Now (instant) | `NOW()` / `CURRENT_TIMESTAMP` | `NOW()` / `UTC_TIMESTAMP()` | `datetime('now')` (UTC) | `SYSUTCDATETIME()` |
| Add 1 day | `d + INTERVAL '1 day'` | `DATE_ADD(d, INTERVAL 1 DAY)` | `datetime(d,'+1 day')` | `DATEADD(day,1,d)` |
| Difference | `b - a` (interval) | `TIMESTAMPDIFF(unit,a,b)` | `julianday(b)-julianday(a)` | `DATEDIFF(unit,a,b)` |
| Truncate to month | `DATE_TRUNC('month',d)` | `DATE_FORMAT(d,'%Y-%m-01')` | `date(d,'start of month')` | `DATETRUNC(month,d)` (2022+) |
| Extract part | `EXTRACT(YEAR FROM d)` | `YEAR(d)` | `strftime('%Y',d)` | `DATEPART(year,d)` |
| Format | `TO_CHAR(d,'YYYY-MM-DD')` | `DATE_FORMAT(d,'%Y-%m-%d')` | `strftime('%Y-%m-%d',d)` | `FORMAT(d,'yyyy-MM-dd')` |
| Parse | `TO_TIMESTAMP(s,'...')` | `STR_TO_DATE(s,'...')` | `datetime(s)` | `TRY_CONVERT`/`PARSE` |
| Week start | Monday (ISO) | Mode-dependent, Sunday default | `strftime('%W')`, Monday | `DATEFIRST`-dependent |

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `WHERE DATE(ts) = :day` | Not sargable, and uses the session zone | Instant range in the chosen reporting zone |
| Storing a UTC offset instead of a zone name | Offsets do not survive DST | Store the IANA zone name |
| Storing a birthday as a timestamp | Shifts a day for users in other zones | `DATE` |
| `days / 365.25` for age in years | Wrong across leap years for some dates | Engine age function, or month-day comparison |
| `INTERVAL '24 hours'` for "next day" | Off by an hour across DST transitions | `INTERVAL '1 day'` when you mean the calendar day |
| Calendar year with ISO week number | Early-January rows land in the wrong year | Pair `ISOYEAR` with `WEEK` |
| Cron at 02:30 local | Skipped or duplicated on transition days | Schedule in UTC |
| Mixing `TIMESTAMP` and `TIMESTAMPTZ` columns in one schema | Every comparison reinterprets one of them without warning | One convention per database, enforced in review |
| Relying on the server's `localtime` | Server zone is infrastructure, not a user preference | Pass the user's zone explicitly |
