# Shortcuts, Panels, and Click Paths

Canvas work is delivered as instructions someone else executes, so every canvas answer names the panel, the field, and the key. "Turn on clip content" is a search task; "Design tab → frame properties → check `Clip content`" is an instruction.

Section names in the panels are stable. The exact slot of a toggle inside a section's advanced settings (`…`) moves between releases — when it is not where this says, `Cmd/Ctrl + /` and type the command name.

## The Instruction Format

`Select <what> → <panel> → <section> → <field> = <value>`, one line per click, in the order they are clicked. A key replaces the line when one exists.

- Left panel: Layers, Assets, Pages. Right panel: Design tab (properties of the selection), Prototype tab (interactions), Dev Mode inspect.
- Name the selection first. Half of "it does not work for me" is the wrong node selected — the child instead of the set, the text node instead of its container.
- Mac keys first, then Windows: this file writes `Cmd/Ctrl` and `Alt/Option` so the instruction can be pasted to either.

## Shortcuts by Task

| Task | Keys | Note |
|---|---|---|
| Wrap selection in auto layout | `Shift + A` | Pressed twice it nests a second, empty frame — check the layer tree after |
| Remove auto layout | `Shift + Alt/Option + A` | Leaves children in place at their current positions |
| Frame the selection | `Cmd/Ctrl + Alt/Option + G` | A frame clips, pads and lays out; `Cmd/Ctrl + G` makes a group, which does none of the three |
| Ungroup | `Cmd/Ctrl + Shift + G` | On a frame this destroys its padding and layout settings too |
| Batch-rename a selection | `Cmd/Ctrl + R` | Find-replace plus numbering tokens (→ SKILL.md Core Rules, rule 5) |
| Create component | `Cmd/Ctrl + Alt/Option + K` | Select several first, then right-click → create multiple components |
| Detach instance | `Cmd/Ctrl + Alt/Option + B` | Last resort → SKILL.md Traps |
| Scale instead of resize | `K` | Scales strokes, radii and text with the geometry; dragging a handle does not |
| Flatten to one path | `Cmd/Ctrl + E` | Irreversible in practice; keep the `_source` copy first |
| Outline stroke | `Cmd/Ctrl + Shift + O` | Converts stroke to fill — same geometry change SVG export forces on non-center strokes |
| Copy / paste properties | `Cmd/Ctrl + Alt/Option + C` / `+ V` | Carries fills, strokes, effects and type onto another node without touching its layout |
| Duplicate | `Cmd/Ctrl + D`, or `Alt/Option + drag` | Drag-duplicate then `Cmd/Ctrl + D` repeats the offset |
| Toggle visibility / lock | `Cmd/Ctrl + Shift + H` / `Cmd/Ctrl + Shift + L` | Hiding is not archiving: hidden layers still load and still cost |
| Export selection | `Cmd/Ctrl + Shift + E` | |
| Place image | `Cmd/Ctrl + Shift + K` | Downscale before placing; Figma keeps the original bitmap |
| Comment mode | `C` | Pin to a node, not to coordinates, or the pin drifts after a restructure |
| Dev Mode | `Shift + D` | Seat-gated; check before promising the link |
| Any command by name | `Cmd/Ctrl + /` | The escape hatch when a menu item has moved |
| Hide the UI | `Cmd/Ctrl + \` | Clean screenshots without exporting |
| Zoom: selection / fit / 100% | `Shift + 2` / `Shift + 1` / `Shift + 0` | |

Selection, which is most of the difficulty:

| Task | Keys |
|---|---|
| Step into a child / back to the parent | `Enter` / `Shift + Enter` |
| Deep-select a leaf through groups and instances | `Cmd/Ctrl + click` |
| Measure distance to another layer | Select one, `Alt/Option + hover` the other |
| Select every node sharing a fill, stroke, font or instance | Right-click → `Select all with same …` |

That last one is the sweep behind every rebind and consolidation pass: it is scoped to the current page, so repeat it page by page.

## Where Each Field Lives

| Field | Path | Decides |
|---|---|---|
| Fill / Hug / Fixed | Design tab → auto layout section → the W and H dropdowns | The whole sizing engine (→ SKILL.md Sizing Modes) |
| min / max width and height | Design tab → the W or H field → advanced settings | Clamps that replace breakpoint frames |
| Clip content | Design tab → frame properties, beside the frame's size | Clipped vs silently spilling over siblings |
| Canvas stacking, stroke inclusion in layout | Design tab → auto layout section → advanced settings (`…`) | Overlap order; the 1px border that shifts everything by 2px |
| Absolute position | Design tab → position toggle, on a child of an auto layout frame | Out of flow, constraints become live, stops feeding a Hug parent |
| Constraints | Design tab → Constraints | Inert unless the parent is manual-layout or the child is absolute |
| Max lines and truncation | Design tab → Typography → type settings (`…`) | The ellipsis nobody asked for |
| Vertical trim | Design tab → Typography → type settings (`…`) | Whether the padding set in Figma equals the padding in the build |
| Component properties | Select the set's dashed container → Design tab → Properties → `+` | Properties added on a single variant apply to that variant only |
| Preferred values | Select the instance-swap property → its edit dialog | A two-item dropdown instead of every component in every library |
| Scoping and code syntax | Variables panel → open the variable → Scoping / Code syntax | Where it appears in pickers; the symbol the engineer copies |
| Mode on a frame | Select the frame → Design tab → the collection's mode selector | Children inherit unless explicitly overridden |
| Export rows | Design tab → Export → `+` per row (scale, format, suffix) | One selection satisfying three platforms |
| Ready for dev | Select the frame or section → right-click, or set it in Dev Mode | What engineers see as scope |
| Publish a library | Assets panel → library icon → Publish | Review per component, never Accept all |
| Description and doc link | Select the main component → foot of the Design tab | The one documentation surface that reaches the assets panel and Dev Mode |

## Click Paths for the Operations This Skill Orders

**Drag audit** (30 seconds, before any handoff): select the top frame → drag its right edge from the minimum supported width to the maximum → paste the longest realistic string into every text node → sweep again. Anything that clips, overlaps or forces horizontal overflow is a defect even if the default width is perfect.

**Rebind a raw hex to a variable**: select one offending layer → right-click → `Select all with same fill` → Design tab → Fill → the variable chip beside the swatch → pick the semantic token. Repeat per page, then re-check in every mode that ships, not only Light.

**Convert a variant axis to a boolean property**: select the set's dashed container → Design tab → Properties → `+` → Boolean, name it as the API (`hasIcon`) → select the optional layer inside one variant → bind its visibility to the property → delete the now-redundant variant rows → check a second variant took the change.

**Version a breaking library change**: name a version first (main menu → File → version history) → duplicate the main as `Button v2` → make the change there → publish → migrate consumers file by file → rename the original `_Button (deprecated → Button v2)` so it leaves the picker while live instances keep working.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| The shortcut did nothing | Focus is in a panel field, or the tool is not the move tool | Click empty canvas, press `V`, retry |
| `Shift + A` produced two nested frames | Pressed twice | Delete the inner frame; it carries no property |
| The field named in the instruction is not there | Wrong node selected, or the property does not apply to that node type | Re-select per the instruction's first line; `Cmd/Ctrl + /` and search the command |
| Constraints panel does nothing | Child is in flow inside an auto layout frame | Change sizing modes, or make the child absolute |
| Select-same missed half the layers | The sweep is per page | Repeat on every page before declaring the rebind done |
| Property added to the set is missing on other variants | Added on one variant instead of the dashed container | Add it on the set, then verify on a second variant |
| Resizing an icon thickened its stroke | Dragged the handle instead of scaling | Undo, press `K`, scale |
