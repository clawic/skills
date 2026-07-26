# Date and Time — java.time Without the Classic Bugs

Rule zero: `Date`, `Calendar`, and `SimpleDateFormat` do not appear in new code. They are mutable, thread-unsafe, and month-zero-indexed. `java.time` (8+) fixed all three.

## Pick the Right Type

| Type | Means | Use for |
|---|---|---|
| `Instant` | A point on the UTC timeline | Timestamps, durations, anything stored or transmitted |
| `LocalDate` | A date with no time and no zone | Birthdays, invoice dates, "what day is it for this user" |
| `LocalTime` | A clock time with no date | Opening hours |
| `LocalDateTime` | Date + time, **no zone** — NOT a point in time | Wall-clock inputs before you know the zone; never a storage format |
| `ZonedDateTime` | A point in time in a region's rules, DST-aware | Future appointments, anything a human schedules |
| `OffsetDateTime` | A point in time with a fixed offset | Wire formats, database `timestamptz` |
| `Duration` | Machine time (seconds/nanos) | Timeouts, elapsed measurements |
| `Period` | Human time (years/months/days) | "One month later", ages |

- The decisive question: *if the time zone rules changed tomorrow, should the value change?* A meeting next March should still be at 09:00 local (`ZonedDateTime`); a log entry should not move (`Instant`).
- Storing `LocalDateTime` for an event is the single most expensive date bug: the value is unusable without the zone you did not store.
- Store UTC (`Instant` / `timestamptz`), convert at the edges for display.

## The DST Traps

- `Duration.ofDays(1)` is exactly 24 hours; `Period.ofDays(1)` is "the same clock time tomorrow". On a DST boundary they differ by an hour, and only one of them is what the user meant. Adding a `Period` to a `ZonedDateTime` is DST-aware; adding a `Duration` is not.
- A local time can be **skipped** (spring forward: 02:30 does not exist — `ZonedDateTime.of` moves it forward by the gap) or **ambiguous** (fall back: 01:30 happens twice — the earlier offset wins by default). Resolve explicitly with `withEarlierOffsetAtOverlap()`/`withLaterOffsetAtOverlap()`.
- Recurring jobs at 02:30 local either run twice or not at all, once a year. Schedule in UTC, or accept and document the behavior.
- Zone IDs are IANA names (`Europe/Madrid`), never three-letter abbreviations (`CST` is ambiguous across three zones). `ZoneId.of("America/New_York")`; `ZoneOffset` only when the offset is genuinely fixed.
- The tzdata inside the JDK ages: DST rule changes ship in JDK updates, so a container image pinned for two years can compute the wrong offset for a country that changed its rules. `java -Xshowsettings:properties -version 2>&1 | grep tzdb` shows the bundled version.

## Formatting and Parsing

- `DateTimeFormatter` is immutable and thread-safe — the fix for the `SimpleDateFormat` static-field bug (SKILL.md Traps). Hoist it into a `static final`.
- Pattern letters that bite:
  - `YYYY` is **week-based year**, `yyyy` is calendar year. `YYYY` on 2025-12-29 prints 2026 — an off-by-one-year bug that appears only in the last days of December.
  - `hh` is 12-hour (needs `a`), `HH` is 24-hour. `mm` is minutes, `MM` is months.
  - `DD` is day-of-year, `dd` is day-of-month.
- ISO first: `DateTimeFormatter.ISO_INSTANT`, `ISO_LOCAL_DATE`, `ISO_OFFSET_DATE_TIME`. Custom patterns only for human display.
- Parsing needs the right target: `Instant.parse` wants an offset in the text; `LocalDateTime.parse` refuses one. `DateTimeFormatter.parse(text, LocalDate::from)` when the query is explicit.
- Localized formats: `ofLocalizedDate(FormatStyle.MEDIUM).withLocale(locale)` — and note that JDK 9 switched the default locale data to CLDR, so formatted output changed compared to Java 8 (`migration.md`).
- Lenient vs strict resolution: `withResolverStyle(ResolverStyle.STRICT)` plus `uuuu` (not `yyyy`) rejects impossible dates like 2025-02-30 instead of silently shifting them.

## Arithmetic That Behaves

- Month arithmetic clamps: `LocalDate.of(2025,1,31).plusMonths(1)` is 2025-02-28. Adding one month twice ≠ adding two months; billing logic must decide explicitly.
- `TemporalAdjusters` covers the calendar-speak cases: `lastDayOfMonth()`, `next(DayOfWeek.MONDAY)`, `firstDayOfNextYear()`.
- `ChronoUnit.DAYS.between(a, b)` truncates toward zero and returns whole units; `Period.between(a, b)` gives years/months/days.
- `isBefore`/`isAfter`/`isEqual` express intent better than `compareTo`; `equals` on `ZonedDateTime` compares the zone too, so the same instant in two zones is not `equals` but `isEqual` says true.
- All `java.time` types are immutable: `date.plusDays(1)` returns a new value. Ignoring the return value is a no-op bug the compiler will not flag.

## Clocks and Testability

- Never call `Instant.now()` deep inside business logic. Inject a `Clock` and call `Instant.now(clock)`; tests use `Clock.fixed(instant, zone)` and `Clock.offset(...)`.
- `LocalDate.now()` uses the system default zone — the reason a date-based test passes in Madrid and fails on a UTC CI runner (`debug.md`). Pass the zone: `LocalDate.now(clock)`, with the configured `timezone` when the user set one (SKILL.md Configuration).
- Elapsed time uses `System.nanoTime()` (monotonic), never `Instant.now()` differences: NTP can step the wall clock backwards mid-measurement (`performance.md`).
- `Instant` has nanosecond precision in the API, but the actual resolution depends on the OS and JDK version — a `Instant.now()` captured on JDK 8 truncates to milliseconds while JDK 9+ gives microseconds, so round-tripped equality can fail after an upgrade.

## Storing and Transporting

- Database: `timestamptz`/`TIMESTAMP WITH TIME ZONE` ↔ `Instant` or `OffsetDateTime`; plain `DATE` ↔ `LocalDate`. Never map a zoned instant onto a naive column.
- JSON: serialize as ISO-8601 strings. Jackson writes `Instant` as a decimal epoch by default until you disable `WRITE_DATES_AS_TIMESTAMPS` and register the JSR-310 module (`serialization.md`).
- Epoch values need their unit stated: seconds vs milliseconds vs microseconds silently differ by 1000×, and the resulting date is either 1970 or the year 55000 — a good sanity assertion.
- Legacy boundaries: `Date.toInstant()`, `Date.from(instant)`, `Timestamp.toLocalDateTime()`. Note `java.sql.Timestamp.equals(Date)` is asymmetric — never mix the two in a collection.
