# Auditing an Inherited or Imported File

Inheriting a file is a forensic job before it is a design job. Editing first is how you discover, three days in, that half the components were detached and the greys are 14 near-identical values.

## Freeze Before Touching

1. Duplicate the file. The duplicate is the working copy; the original stays untouched as the reference.
2. Name a version in the original: "pre-audit baseline, <date>". Named versions are the only reliable rollback.
3. Note the version-history retention on the plan — on the lowest tiers the window is short, and a snapshot duplicate is the real safety net.
4. Do not accept pending library updates yet. They will confuse the inventory with changes you did not make.

## Inventory Pass

Answer these before proposing anything. Most are one plugin run or one REST-side query.

| Question | Why it decides the plan |
|---|---|
| How many pages, and which are actually live? | Dead pages are the cheapest win in an old file — archiving them shrinks everything downstream |
| How many main components, and how many are unused? | Unused mains bloat the picker and hide the real system |
| How many detached instances? | Each is a fork that has already drifted |
| How many colors are actually distinct? | 14 greys means no token system, just habits |
| Styles, variables, or both? | Decides whether this is a migration or a cleanup |
| Are libraries linked, and are they consumed? | Unlinking unused libraries is the cheapest speed win |
| How many text nodes are unstyled? | Measures how far the type ramp actually reaches |
| Layer-name quality on the delivery pages | Predicts handoff cost directly |
| When was it last opened by whom? | An abandoned file may not be worth rescuing at all |

## Rescue or Rebuild

Rebuild when two or more hold:

- Under roughly a quarter of layers sit inside components.
- Structure is manual layout throughout, with no auto layout to build on.
- The color set has no discernible system and the screens disagree with each other.
- The file is a Sketch or XD import that was never restructured.
- Nobody remembers the intent, and the screens contradict the shipped product.

Rescue when the bones exist: auto layout present, a component set that mostly holds, colors clustering into an obvious palette. Rescue is cheaper than it looks once the inventory is done; rebuild is far more expensive than it looks once stakeholders start comparing.

## Cleanup Order

Each step makes the next cheaper. Publish or checkpoint between steps.

1. **Archive**: move dead pages to `Archive`, delete nothing yet.
2. **Rename**: batch-rename layers on the live pages against the naming convention. This alone makes every later step legible.
3. **Consolidate color**: cluster near-duplicate values, pick survivors, sweep with select-same-fill. Do this before building tokens, or the token set inherits the mess.
4. **Build primitives, then semantics**: name after role, never after color.
5. **Rebind components** one set at a time, checking a real screen after each.
6. **Rebind screens**, then sweep for remaining raw hex and typed spacing.
7. **Reattach detached instances** by swapping them to the component; overrides survive where internal layer names match.
8. **Delete** only what the inventory proved unused, and only after a named version.

## Importing From Another Tool

- Frames, text, and vectors import reasonably. Symbols map to components with varying fidelity, shared styles come across partially, and prototype interactions do not come across at all.
- Auto layout is never inferred from an import. Every imported screen is manual layout and must be rebuilt to be responsive — this is the bulk of the migration cost and it is routinely underestimated.
- Imported text often carries per-node overrides instead of styles. Restyle before building a ramp, or the ramp will not stick.
- Treat an import as a visual reference for a rebuild, not as a file to maintain. The one exception is archival work nobody will edit.

## The Deliverable

An audit that lives in someone's head is not an audit. Put it on the cover page of the working copy:

- What is live, what is archived, what was deleted and when.
- The token decisions taken (which greys survived and why).
- Which components are canonical and which are deprecated.
- What is still broken and deliberately deferred, with the reason.
- The date and the person, so the next inheritor knows how stale it is.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Every edit breaks something elsewhere | Detached instances everywhere | Inventory detachment first; reattach by swapping |
| Token set inherited 14 greys | Tokens built before consolidation | Consolidate values, then rebuild the primitive scale |
| Imported screens are not responsive | Auto layout is never inferred on import | Rebuild layout; treat the import as reference art |
| Rebuild ran three times over estimate | Rebuild chosen without an inventory | Inventory first; rescue is usually viable |
| Cannot roll back a bad cleanup | No named version before the pass | Name versions between every step |
| Nobody trusts the cleaned file | No audit note explaining decisions | Write the cover-page audit |
| Deleting an unused component broke a screen | "Unused" measured in this file only | Check consuming files before deleting anything published |
| Library updates arrived mid-audit | Pending updates accepted during inventory | Defer updates until the inventory is complete |
