# Variables, Modes, and Tokens

Variables are the theming and tokenization layer. Four types (color, number, string, boolean), grouped into collections, each collection carrying one or more modes.

## The Three Layers

```
component/button/bg/default  →  semantic/bg/accent  →  primitive/blue/500  →  #2563EB
```

- **Primitive**: the raw scale. No meaning, no mode variation (usually). `blue/500`, `space/4`, `radius/md`.
- **Semantic**: the meaning layer, where modes live. `bg/surface`, `text/muted`, `border/danger`. This is the only layer a mode should change.
- **Component**: created only where a component genuinely deviates from semantic. Most components should never need this layer.

Rules that keep the chain honest:

- Components bind to semantic, never to primitive. Binding a button straight to `blue/500` means retheming is a find-and-replace across every instance in every file.
- A semantic token with exactly one consumer that never differs across modes is noise. Add it when a second consumer or a second mode appears, not before.
- Primitives do not get modes. If `blue/500` changes per theme, the theming is happening one layer too low and the semantic layer has nothing to do.

## Modes

- A mode retheme is one switch: bind every fill to a mode-bound variable, then set the mode on the top frame. Duplicated Light and Dark frame trees drift within a sprint and force every fix to land twice.
- Frames inherit their parent frame's mode unless explicitly overridden. Setting the mode on one nested card is how you preview Dark inside a Light screen without duplicating anything.
- Mode axes multiply the same way variants do. Theme (2) × brand (3) × density (2) = 12 modes on one collection. Split axes across collections instead: a Theme collection, a Brand collection, a Density collection, each with its own small mode list.
- Mode count per collection is plan-gated and the caps differ sharply by tier: effectively single-mode at the bottom, a small handful in the middle, dozens at org and enterprise level. Resolve `figma_plan` before designing a multi-axis matrix (SKILL.md Core Rules, rule 8).

## Scoping and Code Syntax

- **Scoping** restricts where a variable appears in the picker: a radius token that only offers itself in corner-radius fields, a text color that never shows up as a fill option for shapes. This is the cheapest way to stop misbinding in a large token set.
- **Code syntax** attaches a per-platform name (web, iOS, Android) to a variable, and Dev Mode shows that name instead of the Figma path. `semantic/bg/accent` displays as `--color-bg-accent`, `Color.bgAccent`, `R.color.bg_accent` — engineers copy the real symbol instead of translating.
- Hide implementation detail from consumers by prefixing a collection or group with `_`; it stays usable inside the library and disappears from the consumer's picker.

## What Binds to What

| Type | Binds to | Typical use |
|---|---|---|
| Color | Fills, strokes, effect colors | The entire theming surface |
| Number | Padding, gap, corner radius, stroke weight, dimensions | Spacing scale, radius scale, icon sizes |
| Boolean | Layer visibility, component boolean properties | Feature flags in a mock, theme-specific ornament |
| String | Text content, variant property values | Localized copy previews, driving a variant from data |

- Number variables make a spacing scale change propagate without touching a single layout.
- String variables bound to variant properties let one control switch a whole screen's component states — the basis of prototype state machines.

## Where Variables Still Do Not Reach

Coverage has expanded steadily but has never been total. Before promising a variables-only pipeline, check the current gap list for gradients, effect properties, and full text-style binding. Styles remain the fallback, and a style can itself reference a variable — that hybrid is the correct migration state, not a failure.

Practical split that survives the gaps:

- Color, spacing, radius, stroke weight → variables.
- Gradients, shadows and blurs, type ramps, layout grids → styles, with their color components bound to variables where possible.

## Migrating a Styles-Only File

Order matters; doing it out of order creates a half-migrated file nobody trusts.

1. Inventory the styles. Cluster near-duplicates — a file that accumulated fourteen greys rarely needs more than a handful.
2. Build primitives from the surviving clusters. Do not migrate one style per token — that reproduces the mess in a new system.
3. Build the semantic layer against real usage: read the actual screens and name tokens after what they do (`bg/surface-raised`), not after their color.
4. Rebind components to semantic, one component set at a time, publishing after each.
5. Rebind screens last, using select-same-fill sweeps.
6. Delete the old styles only after a full consumer pass — deleting a style unlinks every instance silently.

## Publishing Variables

- Variables publish with the library and must be accepted by the consumer, same as components. A variable that "does not exist" in a consuming file is nearly always an unaccepted library update.
- Variable scope also gates cross-file availability: a collection not marked for publishing stays local no matter how many times you publish.
- Read-side automation of variables through the REST API is tier-gated; plugin-based token export is the route that works on every plan.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Dark mode looks right except for a few layers | Those layers hold raw hex, not a variable | Select-same-fill sweep, rebind |
| Retheming means editing hundreds of values | Components bound to primitives | Insert the semantic layer, rebind components to it |
| Mode switch does nothing on a nested card | Card frame has an explicit mode override | Reset the frame's mode to inherit |
| Cannot add the mode you need | Plan cap on modes per collection | Split axes into separate collections, or upgrade |
| Variables missing in a consuming file | Library update not accepted, or collection not published | Accept the update; check the collection's publish flag |
| Picker offers 200 irrelevant variables | No scoping | Scope each variable to the fields it belongs in |
| Engineers retype token names by hand | Code syntax not set | Add per-platform code syntax to every published variable |
| Gradient cannot be tokenized | Coverage gap | Keep it as a style whose stops reference color variables |
| Token names describe color, not role | Migrated one-to-one from styles | Rename semantically; `bg/surface` outlives `grey/100` |
