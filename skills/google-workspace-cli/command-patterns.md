# Command Patterns - Google Workspace CLI

## Fast Discovery Loop

```bash
gws --help
gws drive --help
gws schema drive.files.list
```

Run this sequence before first use of an unfamiliar resource — per-API parameter caps and required fields live in the schema, not in your memory.

## Read-Only Patterns

```bash
# Drive: always pass an explicit fields mask — v3 returns only kind,id,name,mimeType by default
gws drive files list --params '{"pageSize":10,"fields":"files(id,name,mimeType,modifiedTime,owners)"}'

# Drive: server-side query beats client-side filtering (quota and context)
gws drive files list --params '{"q":"mimeType='"'"'application/pdf'"'"' and modifiedTime > '"'"'2026-01-01T00:00:00'"'"'","fields":"files(id,name)"}'

# Gmail: list returns id stubs only; q uses the same syntax as the Gmail search box
gws gmail users messages list --params '{"userId":"me","maxResults":20,"q":"from:billing newer_than:7d"}'

# Gmail: fetch headers only — format=metadata costs far less quota and context than full
gws gmail users messages get --params '{"userId":"me","id":"MSG_ID","format":"metadata","metadataHeaders":["From","Subject","Date"]}'

# Calendar: singleEvents:true is mandatory with orderBy startTime
gws calendar events list --params '{"calendarId":"primary","maxResults":10,"singleEvents":true,"orderBy":"startTime"}'
```

## Shared Drive Access

```bash
# Without these flags, shared-drive items 404 or silently vanish from lists
gws drive files list --params '{"supportsAllDrives":true,"includeItemsFromAllDrives":true,"corpora":"allDrives","fields":"files(id,name,driveId)"}'
```

## Write Patterns with Safety

```bash
# Draft request first
gws chat spaces messages create \
  --params '{"parent":"spaces/AAA"}' \
  --json '{"text":"Deploy complete"}' \
  --dry-run

# Apply only after confirmation
gws chat spaces messages create \
  --params '{"parent":"spaces/AAA"}' \
  --json '{"text":"Deploy complete"}'
```

## Reversible Deletion (default choice)

```bash
# Drive: trash, do not delete — files.delete bypasses trash permanently
gws drive files update --params '{"fileId":"FILE_ID"}' --json '{"trashed":true}'

# Gmail: trash, do not delete — messages.delete/batchDelete are unrecoverable
gws gmail users messages trash --params '{"userId":"me","id":"MSG_ID"}'
```

## Pagination and Extraction

```bash
# --page-limit = ceil(expected_objects / pageSize); here: ceil(450/100) = 5
gws drive files list --params '{"pageSize":100,"fields":"files(id,name)"}' --page-all --page-limit 5 | jq -r '.files[]?.name'
```

Never bare `--page-all`; add `--page-delay` on quota-sensitive sweeps.

## Upload and Download

```bash
# Multipart upload
gws drive files create --json '{"name":"report.pdf"}' --upload ./report.pdf

# Export a Google-native doc (caps at 10 MB of exported content)
gws drive files export --params '{"fileId":"FILE_ID","mimeType":"application/pdf"}' --output ./out.pdf

# Non-Google binaries: download, not export — the 10 MB cap does not apply
gws drive files get --params '{"fileId":"FILE_ID","alt":"media"}' --output ./file.bin
```
