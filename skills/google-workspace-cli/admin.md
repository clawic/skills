# Admin — Users, Groups, and Audit Reports

Blast radius warning: every write here is tenant-wide. Admin operations always run through full change control (`change-control.md`), on an explicitly named admin account, never on the default account by accident.

## Prerequisites

- The executing account needs Workspace admin privileges — even read-only Directory calls 403 for regular users (`reason: insufficientPermissions` despite a perfectly scoped token).
- Admin scopes are their own family (`admin.directory.user`, `admin.directory.group`, `admin.reports.audit.readonly`, each with a `.readonly` variant) — request the readonly variant for audit work so the token physically cannot mutate.

## Directory: Users

- `users.list` requires `"customer": "my_customer"` or a `"domain"` — omitting both is the classic 400 (`must specify customer, domain, or userKey`).
- Search: `"query": "email:dev-*"`, `"query": "orgUnitPath='/Engineering'"`, `"query": "isSuspended=true"` — server-side, like Drive's `q`.
- `users.insert` needs `primaryEmail`, `name`, `password`; set `"changePasswordAtNextLogin": true` so the provisioning password never becomes the real one.
- `users.update` follows patch semantics — send only changed fields; sending a partial `name` object can blank the missing subfields, so re-read first (SKILL.md Rule 3).
- Offboarding ladder, least to most destructive: reset sign-in cookies + password → `"suspended": true` (reversible, data intact, licenses still billed) → transfer Drive/Calendar data → delete. A deleted user is restorable via `users.undelete` for 20 days (SKILL.md Per-API Limits); after that the data is gone — suspend first, delete on a timer.
- `aliases.insert` adds receive-addresses; send-as remains a Gmail-side setting per user (`gmail.md`).

## Directory: Groups and Members

- `groups.list` with `"userKey": "<email>"` answers "which groups is this person in" — the offboarding and access-review primitive.
- `members.insert` roles: MEMBER | MANAGER | OWNER; members can be users, other groups (nesting), or external addresses if policy allows.
- Posting permissions, external-member policy, and message moderation live in the separate `groupssettings` service, keyed by group email — Directory creates the group, groupssettings governs it.

## Reports: Audit and Activity

`admin-reports` (alias `reports`) answers "who did what, when":

- `activities.list` with `"applicationName"`: `login` (sign-ins, suspicious activity), `drive` (file access/sharing — the "who opened this doc" question), `admin` (console changes), `token` (third-party app grants). `"userKey": "all"` for tenant-wide.
- Filter server-side: `"filters": "doc_id==<id>"`, `"eventName": "login_failure"`, plus `startTime`/`endTime` RFC 3339.
- Data is retained 6 months, and lag varies by application from minutes to days (documented Reports API behavior) — an audit sweep run at incident time is a lower bound, not the whole truth; note the query time in the evidence.
- `customerUsageReports`/`userUsageReport` give storage and license utilization for capacity reviews.

## Licensing

`licensing.licenseAssignments` moves SKUs between users — pair with the offboarding ladder: suspended users keep consuming licenses; deleted users release them.

## Where Admin Work Routes Elsewhere

- Third-party app access reviews → `token` application in Reports plus OAuth client policy in the Admin console (no full API surface; browser required)
- Identity groups with dynamic membership → `cloudidentity` service, which overlaps Directory groups; prefer Directory unless dynamic-membership features are the point
- Leaver file ownership sweeps → `'email' in owners` queries in `drive.md` before the account is deleted
