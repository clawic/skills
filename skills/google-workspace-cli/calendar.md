# Calendar — Events, Recurrence, Invites, and Time

## Time Handling

- `timeMin`/`timeMax` are RFC 3339 WITH offset (`2026-07-23T00:00:00+02:00`); the window is inclusive-start, exclusive-end.
- Timed events use `dateTime` + `timeZone`; all-day events use `date` (no time part) — mixing the two on update morphs the event type.
- List in the calendar's own zone (`"timeZone"` param) when building human-facing agendas; convert once at the edge, not per event.

## Listing Sanely

```bash
gws calendar events list --params '{"calendarId":"primary","timeMin":"2026-07-23T00:00:00Z","maxResults":50,"singleEvents":true,"orderBy":"startTime"}'
```

- `"singleEvents": true` expands recurring series into instances and is mandatory with `orderBy: "startTime"` (400 otherwise). Defaults: 250, max 2500 (SKILL.md Per-API Limits).
- `calendarId` is `primary`, an email address, or a calendar id from `calendarList.list` — the user's visible calendar set, the first call in any multi-calendar workflow.
- Cancelled instances only appear with `"showDeleted": true` (they carry `status: "cancelled"`) — sync jobs that skip this resurrect deleted meetings.

## Creating Events That Actually Invite People

- `attendees` is an array of `{"email": ...}` — but `sendUpdates` defaults to **none**: the API emails nobody. Pass `"sendUpdates": "all"` when humans should be notified (SKILL.md Traps); `externalOnly` for cross-domain courtesy only.
- Meet links don't appear by adding a field to the body: pass `"conferenceDataVersion": 1` as a param plus a `conferenceData.createRequest` with a unique `requestId`.
- Reminders: events inherit the calendar's defaults unless `"reminders": {"useDefault": false, "overrides": [...]}` — an explicit empty override list is how you create a silent event.

## Recurrence

- `recurrence` is an array of RRULE strings: `"RRULE:FREQ=WEEKLY;BYDAY=MO,WE;UNTIL=20261231T235959Z"` (also EXDATE/RDATE lines).
- Edit one occurrence: get its instance id via `events.instances`, then patch that instance — patching the parent edits the whole series.
- "This and following": truncate the parent's UNTIL to just before the split date, create a new series from the split — there is no single API call for it.
- Instance ids encode the occurrence date; deleting the parent deletes every instance.

## Ids, Import, and Dedupe

- `id` is calendar-local; `iCalUID` is the cross-system identity. `events.import` (as opposed to `insert`) preserves an existing iCalUID — the dedupe path when mirroring events from another system; re-import with the same iCalUID updates instead of duplicating.

## Availability and Sharing

- `freebusy.query` answers "when are these N people free" in one call — busy blocks per calendar, no event details, works across colleagues' calendars you can't read.
- Calendar-level sharing is `acl.insert` (`role`: reader | writer | owner; scope type: user | group | domain) — sharing a calendar, unlike Drive, sends no notification email by default.

## Concurrency

Events carry etags and a `sequence` number; updating from a stale read can 409/412, and attendee clients ignore updates whose sequence didn't advance. Re-read immediately before patch (SKILL.md Rule 3) and carry the fresh `sequence` forward on full updates.
