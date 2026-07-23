# Drive — Files, Queries, Sharing, and Transfer

## The Query Language

`q` is a typed expression language, not free text:

```text
name contains 'report'                          # substring on name
mimeType = 'application/pdf'                    # exact type
mimeType = 'application/vnd.google-apps.folder' # folders ARE files with this type
'1AbC...' in parents                            # children of a folder (by id, not name)
modifiedTime > '2026-01-01T00:00:00'            # RFC 3339
fullText contains 'invoice 4471'                # content search (indexed docs only)
sharedWithMe and trashed = false
```

- `files.list` includes trashed items unless you filter `trashed = false` — counts and sweeps are wrong without it (SKILL.md Traps).
- There is no path API: resolve a path by walking `'<parent-id>' in parents` one level at a time, or search by name and disambiguate by parent chain. Names are not unique (SKILL.md Rule 3).
- Always pass explicit `fields` — v3 defaults to `kind, id, name, mimeType` only: `"fields": "files(id,name,mimeType,modifiedTime,owners),nextPageToken"` (keep `nextPageToken` in the mask or pagination silently stops).

## Shared Drives

Three flags or the items don't exist for you: `"supportsAllDrives": true` on every call, plus `"includeItemsFromAllDrives": true` and `"corpora": "allDrives"` on list. Shared-drive permissions are membership-based (managed on the drive, not per file); item `owners` is absent — the drive owns the files.

## Upload, Download, Export

| Direction | Call | Notes |
|-----------|------|-------|
| Upload binary | `files.create` + `--upload ./file` | Set `parents` in metadata or it lands in root |
| Upload → Google format | same, with target `mimeType` in metadata (e.g. `application/vnd.google-apps.document`) | Server-side conversion on ingest |
| Download binary | `files.get` with `"alt": "media"` | No size cap concerns of export |
| Export Google-native | `files.export` with target `mimeType` | Caps at 10 MB of exported content (SKILL.md Per-API Limits) |

Export format traps: Sheets → `text/csv` exports only the FIRST sheet — per-sheet CSV needs one export per sheet or the Sheets values API (`editors.md`); Docs → pdf/docx/txt/html; Slides → pdf/pptx. Oversized Docs: export per-section or switch target format.

## Sharing and Permissions

`permissions.create` grants `role` (reader | commenter | writer | organizer) to `type` (user | group | domain | anyone).

- `sendNotificationEmail` defaults to **true** for users and groups — a bulk-sharing sweep becomes a mass-email incident; set it `false` unless notifying is the point (SKILL.md Traps).
- `type: anyone` = anyone-with-link; combined with `role: writer` it is a publicly editable file — treat as a change-control-level action.
- Ownership transfer: `transferOwnership: true` on a `role: owner` grant — same-domain only in Workspace; consumer accounts require the recipient to accept an invitation. Files a leaver owns don't transfer themselves: sweep `'email' in owners` before offboarding (`admin.md`).
- Audit who sees what: `permissions.list` per file with `fields` covering `emailAddress, role, type, domain`.

## Copies, Shortcuts, Revisions

- `files.copy` also converts formats (copy a template Doc per record — the mail-merge move, `editors.md`); the copy is owned by the caller, permissions do NOT carry over.
- Shortcuts are stub files (`mimeType: application/vnd.google-apps.shortcut`) with `shortcutDetails.targetId` — dereference before acting or you mutate the stub.
- `revisions.list`/`revisions.get` recover prior binary versions; set `keepForever` on revisions you must retain — Drive prunes old binary revisions on its own schedule.

## Incremental Sync (Changes API)

Polling `files.list` for diffs burns quota and misses deletions. Instead: `changes.getStartPageToken` once, persist it, then `changes.list` with the stored token per run — returns adds, modifications, AND removals; persist `newStartPageToken` after each sweep (`automation.md`).

## Service-Account Gotchas

- A bare service account has its own empty Drive: `files.list` returning nothing is the wrong identity, not an error (`auth-playbook.md`).
- Files the service account creates are owned by it and count against its own storage quota — upload into shared drives or impersonate a user; don't build a corpus inside an identity nobody can log into.

## Idempotency Markers

`appProperties` (private per-app key-values, searchable: `appProperties has { key='processed' and value='true' }`) mark processed files so reruns skip work — the Drive equivalent of Gmail's label marker (`automation.md`).
