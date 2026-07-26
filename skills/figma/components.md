# Components — Sets, Properties, and the Instance Contract

A component is an API. Everything that goes wrong later — unfindable variants, overrides that vanish, engineers rebuilding instead of reusing — traces back to an API defined by accident.

## Where Mains Live

- Main components belong in a dedicated library file, or at minimum a dedicated `Components` page. Mains sitting next to the screens that use them get edited by accident and have no version boundary.
- A name starting with `_` or `.` is hidden from the assets panel and from library publishing. This is the deprecation and work-in-progress mechanism: `_Button (deprecated)` keeps every live instance working while removing it from the picker.
- Keep a `_source` or `_scratch` page for editable originals (unflattened icons, exploration variants) so cleanup never destroys the only copy.

## Naming for the Picker

- Slashes build hierarchy in the assets menu: `Button / Primary / Large` reads as three nested levels. Name for the person searching the picker, not for the layer list.
- Keep property *values* identical across sets that should be interchangeable. Swapping `Icon / Search` for `Icon / Close` preserves overrides only when the layer names inside match.
- Component descriptions and documentation links surface in the assets panel and in Dev Mode. One sentence on when to use the component and one link to the code component is the highest-leverage documentation in the file.
- `Cmd/Ctrl + R` batch-renames a selection with find-and-replace and numbering tokens. Run it before publishing, never after.

## Variant Sets

- One concern per property. `State` (default, hover, active, disabled, focus), `Size` (sm, md, lg), `Emphasis` (primary, secondary, ghost). Fusing two concerns into one value (`Primary-Large`) destroys the reason variants exist.
- The default instance is the first variant in the set — the top-left cell in the set's layout. Reorder deliberately; whatever sits there is what everybody inserts.
- Combinatorial math: `total = product of every axis kept as a variant`. A set carrying four states, three sizes, three emphases, a leading icon and a trailing icon is 4 × 3 × 3 × 2 × 2 = 144. At roughly 40-60 variants the set becomes slow to edit and impossible to browse. Keep the three axes a designer compares in the dropdown and convert the two icon toggles: 4 × 3 × 3 = 36 variants plus 2 boolean properties reaches the same 144 combinations.
- An incomplete matrix does not error. Figma resolves a missing combination to the nearest existing variant, so a set with no `disabled + ghost` cell silently renders `disabled + primary` and nobody sees the gap until code review. Fill every cell, or drop the axis.
- Splitting instead of collapsing is the escape hatch: `Button` and `Icon Button` as separate sets is better than one set with a `hasLabel` variant axis, because the two have different anatomy.
- Interactive components live inside the set: wire `hover` to the hover variant once and the behavior travels everywhere the component is used.

## Properties

Which property type an axis belongs in, with its cost and its trap: the Component Property Types table in `SKILL.md`. The mechanics of running them:

- Set **preferred values** on every instance-swap property. Without them the swap dropdown lists every component in every linked library, and people pick wrong.
- Property order in the panel is the order the engineer reads. Put the property that changes most often first.
- **Exposed nested instances** surface a child component's properties on the parent. Powerful and brittle: the exposure breaks when the child is renamed or restructured. Lock a child's property names before nesting it widely.
- Figma has no true slot. The working pattern is an instance-swap property whose default is a tiny empty placeholder component (`.slot`), letting consumers drop in arbitrary content while keeping the parent's layout.
- One boolean can drive several layers at once. That is the correct build for "the leading icon and its gap appear together", and the reason two properties must never target the same layer — the last one applied wins and the other reads as broken.
- Property names travel: they surface in Dev Mode and become prop names through Code Connect. Name them as the API you want (`hasIcon`, `showBadge`, `label`), not as the layer (`Icon 2 visible`).
- Renaming a property breaks every instance's stored value for it, silently resetting to the default. Rename before the second consumer exists, or ship it as a `v2` set.
- A property applied to a layer inside a variant applies to that variant only. Add properties on the set (the dashed container) so every variant carries them, and check a second variant before assuming it took.

## Overrides

- **Reset overrides** re-syncs an instance to its main. **Push overrides to main** pushes the local tweak up to every instance. Know which you mean before clicking; the second is irreversible without version history.
- Overrides survive a swap between components whose internal layer names match. This is why consistent internal naming across an icon set matters more than the icon names themselves.
- Overrides do not survive a main-component restructure. Renaming or reparenting a layer inside the main drops every override that pointed at it — which is why library restructures need versioning (`v2`), not in-place edits.
- Detaching is a smell with three legitimate uses: one-off marketing artwork, a fork you are deliberately turning into the next version, and salvaging from a library you no longer have access to. Every other detach means a variant or property is missing — add it.

## Publishing a Set

- Publish the component set (the dashed container), not the loose variants, so consumers see one menu entry instead of twelve.
- A component whose only consumer is one screen is not a component yet. Promote on the second use, not the first.
- Before publishing: description written, preferred values set, default variant correct, layer names semantic, all sizing modes verified with a drag sweep.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Editing one variant lags for seconds | Set past its size ceiling | Split the set, or convert uncoupled axes to properties |
| Nobody can find the right variant | Two concerns fused in one property | Split the property; rename values to the concern they express |
| Everyone inserts the wrong state | Default variant is not the first cell | Reorder the set |
| Overrides vanish after a library update | Main was restructured in place | Restore from version history; ship the change as `v2` next time |
| Swap dropdown lists hundreds of components | No preferred values on the instance-swap property | Set preferred values to the intended set |
| Overrides lost when swapping icons | Internal layer names differ between the two icons | Standardize the layer name inside every icon in the set |
| Exposed nested properties disappeared | Child component renamed or reparented | Restore the child's structure, then freeze its property names |
| Instances everywhere are detached | A needed variant or property never existed | Add it, then re-attach by swapping detached copies to the component |
| Deprecated component still clutters the picker | Unpublished instead of renamed | Prefix the name with `_` so live instances keep working |
