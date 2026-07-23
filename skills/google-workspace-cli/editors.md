# Sheets, Docs, Slides — The Editor APIs

One mental model for all three: content lives in a structured document; simple cell values have a dedicated Sheets values API; everything else (formatting, structure, text surgery) is a `batchUpdate` with an array of typed requests whose `replies` array aligns index-for-index with the requests.

## Sheets: Reading Values

```bash
gws sheets spreadsheets values get --params '{"spreadsheetId":"ID","range":"'"'"'Q3 Data'"'"'!A1:C10"}'
```

- Ranges are A1 notation; sheet names with spaces need single quotes inside the range string. `spreadsheetId` comes from the URL; `sheetId` (the `gid=` number) is a different identifier used by `batchUpdate` GridRanges — don't swap them.
- `valueRenderOption`: `FORMATTED_VALUE` (what the UI shows — locale-dependent strings) vs `UNFORMATTED_VALUE` (raw numbers) vs `FORMULA`. Automation should read `UNFORMATTED_VALUE`; a script parsing `"1.234,56"` is a locale bug waiting.
- Dates arrive as serial numbers under `UNFORMATTED_VALUE` (`dateTimeRenderOption: "SERIAL_NUMBER"`, days since 1899-12-30) — convert deliberately.
- Many ranges: `values.batchGet` with a `ranges` array — one call instead of N.

## Sheets: Writing Values

- `values.update` and `values.append` REQUIRE `valueInputOption`: `USER_ENTERED` parses input as if typed (strings become dates/numbers, `=SUM(A1:A9)` becomes a live formula); `RAW` stores the literal string. Sending a formula with `RAW` stores dead text that starts with `=`.
- `values.append` finds the current "table" boundary from the given range and writes below it — leading empty rows shift where data lands; anchor the range at the table's real header.
- Structure and formatting (add sheets, resize, cell formats, data validation) are `spreadsheets.batchUpdate` requests (`addSheet`, `repeatCell`, `updateDimensionProperties`...), addressed by `sheetId` + GridRange (zero-based, end-exclusive).
- Size ceiling: 10 million cells per spreadsheet (SKILL.md Per-API Limits) — bulk pipelines that append forever hit it; rotate spreadsheets by period.

## Docs: Index Arithmetic

- `documents.get` returns the body as a tree of structural elements with character `startIndex`/`endIndex`.
- `documents.batchUpdate` `insertText`/`deleteContentRange` work on those indexes — and every insert shifts all later indexes. Apply edits in descending index order within one batch (documented Docs guidance), or compute offsets as you go.
- For template filling, skip index arithmetic entirely: `replaceAllText` with `{{placeholder}}` tokens.

## Slides: Template Filling

`presentations.batchUpdate` mirrors Docs; `replaceAllText` plus per-element `objectId` operations. The standard mail-merge pipeline:

1. `files.copy` the template Deck/Doc per record (`drive.md`)
2. `replaceAllText` each `{{token}}` with the record's values
3. `files.export` to PDF (10 MB of exported content cap) or share with `sendNotificationEmail: false`

Same three steps power contract generation, report packs, and certificate runs — the highest-leverage cross-service pattern in the suite.

## Traps Local to the Editors

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| `values.update` without `valueInputOption` | The API rejects the call — the param has no default | Choose `USER_ENTERED` or `RAW` explicitly |
| Formulas written with `RAW` | Stored as inert text beginning with `=` | `USER_ENTERED` |
| Parsing `FORMATTED_VALUE` in scripts | Locale-formatted strings (`1.234,56`) | `UNFORMATTED_VALUE` + explicit formatting at the edge |
| Docs edits in ascending index order | Each insert invalidates later indexes | Descending order, or `replaceAllText` |
| Exporting a multi-sheet file to CSV | `text/csv` exports only the first sheet | Per-sheet export, or read via the values API |
