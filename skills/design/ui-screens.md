# UI Screens — Forms, Tables, Dashboards, States

Component interaction patterns live in the `ui` skill; this file covers the visual execution decisions that make screens read correctly. Units follow `platform` (px web, pt iOS, dp Android).

## Forms

- Single column. Multi-column forms break the vertical scan and cause skipped fields; the only exception is tightly coupled pairs (city + zip) on one row.
- Labels above fields, left-aligned — survives narrow widths and localization growth; placeholder text is NOT a label (vanishes on focus, fails recall).
- Field width signals expected content: zip ~6ch, name ~30ch. Full-width everything hides what's expected.
- Spacing per Core Rule 3: 8px label-to-field, 24px field-to-field, 48px section-to-section.
- Inputs 16px font minimum — below 16px iOS Safari zooms the viewport on focus.
- Errors: inline under the offending field, red text + icon, field border red; keep the label visible. A summary toast alone strands the user hunting.
- One primary button per form, aligned to the field column's line, action-verb label ("Create account", not "Submit"). Secondary action styled as ghost/link, never a second filled button.

## Tables

- Numbers right-aligned with tabular figures; text left-aligned; headers align with their column's content. Center-aligned columns scan worst — avoid.
- Row height: comfortable 48-56px, dense 32-40px (`base_unit` 4 mode); pick one per table.
- Zebra stripes OR row borders, never both; with generous row height, whitespace alone separates fine.
- Numeric precision is design: align decimal points, same decimals per column, thousands separators. 1,204.5 over 1204.50000 in the same column screams unreviewed output.
- Long tables: sticky header, first column pinned if it's the row identity.
- Row actions appear on hover/focus on pointer devices but stay always-visible on touch (`platform`).

## Dashboards

- One rank-1 metric per screen — the number the user opens the dashboard for — at display size (31-39px). Everything else supports it.
- KPI cards: value large (25-31px), label small and muted above or below, trend delta with direction glyph + color (never color alone, Core Rule 7).
- Group charts by question, not by chart type; a 2x2 of related charts beats 8 unrelated tiles.
- Full-width layout, 24px outer margins, cards on the grid with equal gutters (`layout.md`).
- Refresh/timestamp visible when data can be stale: "as of 09:41" is rank-3 metadata, small and muted, but present.

## The Four Non-Happy States

Design these before polishing the happy path — they ship broken otherwise (SKILL.md Traps):

| State | Rule |
|---|---|
| Empty | Teach, don't apologize: what this area will show + the action that fills it. Never a bare "No data" |
| Loading | Skeleton in the final layout (prevents shift); spinner only for sub-second or indeterminate whole-screen waits |
| Error | What failed, in user terms + retry action. Keep the layout; don't collapse to a blank error page for a partial failure |
| Overflow | Decide truncation per element: ellipsis + tooltip for identifiers, line-clamp for descriptions, "+N more" for chips. Test with 3x expected content length and German-length words |

## Buttons and Actions

- Hierarchy: one filled primary, outlined/ghost secondary, text-only tertiary. Two filled buttons side by side = two rank-1s, fails the squint test.
- Destructive actions: red filled only at the confirmation step; the entry-point button stays neutral with a red icon/label.
- Touch targets 44pt/48dp minimum; visual element may be smaller with padded hit area.
- Disabled buttons need a reason nearby (helper text under the form) — a mute button with no explanation is a dead end.

## Icons and Imagery

- One icon set, one stroke weight, one optical size grid (usually 24px). A single off-set icon reads as broken.
- Icons without labels only for the universal ten (search, close, settings, back...); everything else gets a text label — mystery meat costs more than the space saved.
- Avatars/thumbnails: fixed aspect, object-fit cover, letter-fallback with deterministic background from the name hash.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Placeholder-as-label forms | Label gone while typing; autofill collisions | Label above, placeholder for format example only |
| Modal for everything | Context lost, stacking modals trap users | Inline expansion or dedicated page for multi-step work |
| Toast for critical errors | Auto-dismisses before it's read | Inline persistent error at the failure site |
| Truncating without tooltip/expansion | Data becomes unrecoverable in the UI | Every truncation has a full-content affordance |
| Same visual weight for Cancel and Confirm | Squint test fails; misclicks on destructive flows | Primary filled, secondary ghost — always |
