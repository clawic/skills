# Gmail — Search, Send, Labels, and Bulk Mail

## Search That Finds What You Mean

`q` uses the same syntax as the Gmail search box: `from:`, `to:`, `cc:`, `subject:`, `label:`, `has:attachment`, `filename:pdf`, `larger:5M`, `is:unread`, `newer_than:7d`, `older_than:1y`, `after:2026/01/01`, `rfc822msgid:<id>`.

- Search excludes spam and trash by default. `"includeSpamTrash": true` or `in:anywhere` widens it — the difference between "no results" and "it was in spam" during investigations.
- `newer_than:` beats `after:` for rolling automation windows — no date arithmetic in the script.
- Server-side `q` always beats client-side jq filtering: quota and context both (`quotas.md`).

```bash
gws gmail users messages list --params '{"userId":"me","maxResults":20,"q":"from:billing newer_than:7d has:attachment"}'
```

## Message Anatomy and Format Tiers

`messages.list` returns only `id` + `threadId` stubs. `messages.get` format tiers, cheapest first:

| Format | Returns | Use for |
|--------|---------|---------|
| `minimal` | ids, labels, snippet | existence and label checks |
| `metadata` + `metadataHeaders` | chosen headers only | triage sweeps — far less quota and context than full |
| `full` | parsed MIME tree | body extraction, attachment ids |
| `raw` | base64url RFC 2822 blob | forensics, re-import |

Body data in `full` payloads is base64url-encoded per MIME part; multipart messages nest — walk `payload.parts` recursively and pick the `text/plain` or `text/html` leaf.

## Threads and Real Replies

- `threads.get` returns every message in a conversation in one call — cheaper than N × `messages.get` when you need the whole exchange.
- A reply only threads in recipients' clients when the outgoing message sets `In-Reply-To` and `References` headers to the original `Message-ID` header AND shares the `threadId` — the API accepts a bare `threadId`, but other mail clients thread by headers.

## Sending

`messages.send` takes `raw`: a base64url-encoded RFC 2822 message you build yourself (headers `From`, `To`, `Subject`, `MIME-Version`, `Content-Type`; attachments as multipart MIME inside it).

- base64url, not standard base64 — `+` and `/` must become `-` and `_`, or the API rejects or mangles the message.
- Limits (→ SKILL.md Per-API Limits): send = 100 quota units against a 250 units/second budget, so a mail blast saturates at roughly 2 sends/second/user (250/100 = 2.5); daily cap 2,000 (Workspace) / 500 (consumer).
- Aliases: the `From` header may only use addresses registered under Settings send-as; anything else is silently rewritten to the primary address.

## Drafts as Mail's Dry-Run

`--dry-run` previews the request, not the delivery. For anything user-facing, create drafts (`drafts.create` with the same `raw`), let a human open them in Gmail, then `drafts.send`. This is the approval gate `change-control.md` expects for bulk mail.

## Labels — the Idempotency Backbone

- System labels (`INBOX`, `UNREAD`, `SPAM`, `TRASH`, `STARRED`) vs user labels; nesting is name-based: `Automation/Processed`.
- Filter lists by label: `"labelIds": ["Label_123"]` — label ids, not names; resolve once via `labels.list` and cache in `command-log.md`.
- `messages.modify` with `addLabelIds`/`removeLabelIds` mutates one message; `messages.batchModify` takes up to 1,000 message ids per call — the bulk-labeling workhorse and the standard "mark as processed" primitive for sweeps (`automation.md`).

## Trash vs Delete

`messages.trash` is reversible (Gmail purges trash after ~30 days); `messages.delete`/`batchDelete` are permanent, bypass trash, and require the full `https://mail.google.com/` scope — `gmail.modify` cannot do it (scope table in `auth-playbook.md`). Default to trash (SKILL.md Rule 7).

## Attachments

Attachment bytes are NOT in `messages.get` responses — the `full` payload carries an `attachmentId` per part; fetch bytes with `messages.attachments.get`, then base64url-decode. Filename and mimeType come from the part headers, not the attachment call.

## Filters and Settings

`settings.filters.create` (criteria → action: add/remove labels, forward, never spam) automates at Google's edge instead of polling. Settings resources need their own scopes (`gmail.settings.basic`; forwarding/delegation are in the sharing tier) — a working mail token still 403s on settings.

## Incremental Sync

Don't re-list the mailbox: `history.list` with `startHistoryId` from a previous fetch returns only changes since. A 404 on the historyId means it expired — fall back to a full list and store the new baseline (pattern in `automation.md`).
