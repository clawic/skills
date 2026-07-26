# Libraries — Publishing, Versioning, and Adoption

A library is a distribution channel with consumers you cannot roll back. Everything in this file exists because a library edit reaches every consuming file the instant Publish is pressed.

## Library Architecture

- **Foundations library**: variables (color, spacing, radius, type ramp) and nothing else. Product files that need tokens but not components link this one alone and stay fast.
- **Component libraries**: one per product surface (marketing, app, admin) or per brand. Each links the foundations library.
- **Icon library**: separate. Icon sets churn on a different cadence from components and are consumed by teams that need nothing else.

A single mega-library makes every linked file pay the full load cost and turns every edit into org-wide blast radius. Federate when a second team needs a different release cadence — that trigger, not file size, is the signal.

## The Publish Cycle

1. Finish the change on a branch (where available) or in a scheduled edit window.
2. Write the release note in the publish dialog. "Updated components" is worthless six weeks later; name what moved and what breaks.
3. Publish only the components and variables that changed — the dialog lets you deselect, and a smaller diff makes consumer review possible.
4. Announce in the team channel with the release note and the migration action, if any.

On the consumer side:

- A library update surfaces as a per-component `Review` list. Review per component. `Accept all` on a structural change silently rewires overrides across every screen in the file.
- Before accepting a structural change, open one real consuming screen and accept there first. If overrides survive, accept the rest.
- Accepting is not reversible in place — recover through version history, which is why naming a version before a bulk accept is worth the ten seconds.

## Versioning a Breaking Change

An in-place restructure of a main delivers breakage everywhere simultaneously. The versioned path costs one extra component and removes the coordination problem:

1. Duplicate the component: `Button` → `Button v2`. Restructure the copy.
2. Publish `v2` alongside the original. Nothing breaks.
3. Migrate consumers file by file, using instance swap. Overrides survive where internal layer names match.
4. Deprecate the original: rename it `_Button (deprecated → Button v2)`. It disappears from the assets panel; live instances keep working.
5. Delete only after the adoption metric shows zero remaining instances.

What counts as breaking: renaming or reparenting a layer that overrides point at, removing a variant, renaming a property, changing the auto layout direction of the root frame. What does not: adding a variant, adding a property with a sensible default, changing a bound token's value.

## Branching

Where the plan includes it, branching stages library changes behind a merge review: branch the file, edit, request review, merge. The reviewer sees a visual diff per component.

- Use it for anything that touches more than one component set.
- Comments made on a branch stay on the branch until merge — reviewers who comment on the main file are commenting on the old version.
- Without branching, the substitutes are: duplicate the file as a working copy, name a version before and after the edit, and schedule edits so no two people restructure the same set in the same day.

## Swapping and Rebranding

- Swap library replaces every instance from library A with the matching component in library B, matched by name. This is the rebrand and consolidation tool; it is also the reason naming discipline pays off years later.
- A swap that silently misses components means the names diverged. Run a name diff between the two libraries before the swap, not after.

## Adoption and Cleanup

- Library analytics (tier-gated) report insertions per component and detach rate. A high detach rate on one component is a specification: the variant people need does not exist.
- Zero-insertion components after two release cycles are dead weight. Deprecate them; the picker is a shared resource.
- There is no native "find unused mains" — it takes a plugin or a REST-side audit that walks every consuming file's component instances.
- Detached instances across consuming files are the other cleanup target: each one is a fork that will drift.

## Library Hygiene

- Cover page first: the first frame on the first page becomes the thumbnail, which is how anyone finds the library in recents.
- Page order that survives scale: `Cover`, `Foundations` or `Tokens`, `Components`, `Patterns`, `_Source`, `Archive`.
- Every published component carries a one-sentence description and a documentation link. Both surface in the assets panel and in Dev Mode.
- Keep an `Archive` page rather than deleting: deleted components break instances, archived ones just leave the picker.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| One library update broke overrides in ten files | Main restructured in place | Restore from version history; reship as `v2` |
| Consumers never update | No release note, no announcement | Write the note in the publish dialog; announce with the migration action |
| Every linked file is slow to open | Mega-library linked everywhere | Split foundations out; unlink libraries a file does not use |
| Swap library missed half the instances | Component names diverged between libraries | Name-diff the two libraries and align before swapping |
| A component is detached everywhere | The needed variant does not exist | Read the detach rate, add the variant, then swap detached copies back |
| Nobody can find the library | No cover, generic file name | Cover frame first page; name the file for the surface it serves |
| Deleting an old component broke live screens | Deletion unlinks instances | Rename with `_` prefix instead; delete only at zero instances |
| Two people restructured the same set today | No branching, no edit window | Branch where available; otherwise schedule and announce edits |
