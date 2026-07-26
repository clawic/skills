---
name: Figma
slug: figma
version: 1.0.2
description: 'Builds and debugs Figma files: auto layout, components and variants, variables and modes, libraries, prototypes, and Dev Mode handoff. Use when a frame will not resize, text clips or overflows, a variant set lags, a library update breaks instances, dark mode or theming needs modes instead of duplicated screens, engineers say the UI breaks on resize, icons export blurry, a file opens slowly, an inherited file needs auditing, or work has to be scripted through plugins, the REST API, or the Dev Mode MCP server. Covers component properties, token naming, Code Connect, export densities, FigJam and Slides. Not for piping tokens into platform code (design-tokens), prototyping beyond Figma (prototyping), visual design judgment (design), or the handoff process around the file — spec documents, redlines, review rituals (design-handoff).'
homepage: https://clawic.com/skills/figma
changelog: "Display name shown correctly"
metadata:
  clawdbot:
    emoji: 🎨
    os:
    - linux
    - darwin
    - win32
    displayName: Figma
    configPaths:
    - ~/Clawic/data/figma/
---

User preferences and observed context live in `~/Clawic/data/figma/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/figma/` or `~/clawic/figma/`), move it to `~/Clawic/data/figma/`.

## When To Use

- Building or restructuring a file: auto layout, component sets, variables and modes, library architecture
- Debugging a file: a frame that will not resize, clipped text, a laggy variant set, a file that takes a minute to open
- Preparing handoff: Dev Mode annotations, Code Connect, Ready-for-dev scope, the sad-path states engineers otherwise invent
- Theming and tokens inside Figma: dark mode, brand and density modes, alias chains, plan-gated mode limits
- Automating: plugins, the REST API, the Dev Mode MCP server, bulk renames, unused-component audits
- Not for visual design judgment (design), token pipelines into platform code (design-tokens), or prototypes that outgrow Figma (prototyping). Dev Mode annotations, Code Connect and Ready-for-dev scope live here because they are built inside the file; the spec document, the redline artifact and the review ritual around it are design-handoff

Mode: **act-as** through the API and plugin surfaces, **advise** on the canvas. Figma has no CLI, so canvas work is delivered as named panel fields and shortcuts the user executes, in click order (`shortcuts.md`); anything scriptable is delivered as plugin code or REST calls.

## Quick Reference

| Situation | Play |
|---|---|
| A frame won't grow or shrink | Fill child inside a Hug parent on the same axis — the size is undefined and Figma freezes it. Fix the outermost container first, then work inward (→ Sizing Modes) |
| Text clips, wraps wrong, or shows an unexpected ellipsis | Text node is Fixed size, or Max lines is set; switch to Auto height and let the parent Hug → `text.md` |
| Engineers say "it breaks on resize" | Absolute positioning where auto layout belongs; the screenshot was fine, the tree was not → `auto-layout.md` |
| Dark mode, brand themes, or density variants needed | One tree, a second mode on the color collection, every fill bound to a semantic alias → `variables.md` |
| A variant set lags when edited, or nobody can find a variant | Past the combinatorial ceiling; move uncoupled axes to boolean, text, and instance-swap properties (→ Component Property Types) → `components.md` |
| A library update broke consuming files | Review per component, never Accept all; version breaking changes instead of editing a main in place → `libraries.md` |
| Handoff churn: engineers rebuild instead of reusing | Components have no code identity — map them with Code Connect and annotate what code cannot infer → `dev-mode.md` |
| Prototype motion crossfades instead of moving | Smart Animate found no name-and-hierarchy match across the two frames → `prototyping.md` |
| Icons look blurry, misaligned, or export at the wrong size | Stroke alignment plus off-pixel bounds; export from a fixed keyline frame → `vectors.md` |
| Assets needed at @2x/@3x, mdpi→xxxhdpi, or as clean SVG | Export settings and density presets per platform → `export.md` |
| The file takes forever to open or edit | Rank the causes: raster images, vector node count, effects, layer count, fonts → `performance.md` |
| The same edit has to happen on 50 layers | 10+ repetitions justifies a plugin; read-only audits belong on the REST API → `plugins.md` |
| A pipeline needs the file's structure, tokens, or renders | Targeted node fetches, not whole-file pulls; webhook on library publish → `rest-api.md` |
| A feature seems missing from the UI | Plan gate, not a bug — modes, branching, Dev Mode, analytics and the Variables API are all tiered → `collaboration.md` |
| Contrast, focus order, or touch targets need speccing | Annotate what the visual layer cannot carry → `accessibility.md` |
| Speccing for iOS, Android, or a foldable | Densities, safe areas, system UI and touch minimums differ from web → `mobile.md` |
| Inherited a messy file, or importing from Sketch/XD | Inventory before editing; freeze a named version first → `audit.md` |
| A workshop, flow diagram, deck, site, or generated draft is the deliverable | FigJam, Slides and the newer publishing and generative surfaces are separate editors with their own object models and seat gates → `surfaces.md` |
| The fix is a canvas operation the user has to perform | Deliver panel, section, field and key in click order — never "find the setting" → `shortcuts.md` |
| Anything else | Rebuild the smallest failing case in a fresh file with two frames. If it behaves there, the bug is this file's structure, not Figma |

Depth on demand: `auto-layout.md` sizing engine · `components.md` sets and properties · `variables.md` tokens and modes · `libraries.md` publishing and versioning · `dev-mode.md` handoff and Code Connect · `prototyping.md` flows and motion · `text.md` type mechanics · `vectors.md` icons and paths · `export.md` assets and densities · `performance.md` file health · `plugins.md` choosing and writing plugins · `rest-api.md` automation · `collaboration.md` plans, permissions, review · `accessibility.md` speccing a11y · `mobile.md` platform rules · `audit.md` rescuing a file · `surfaces.md` FigJam, Slides and the newer editors · `shortcuts.md` shortcuts, panel paths and click order.

## Core Rules

1. **Fill by default; Fixed only for genuinely fixed dimensions.** Every typed px is a future bug report. Fixed is right for icon frames, avatars, and rail widths; everything else is Fill with min/max clamps. A Fill child in a Hug parent on the same axis has no defined size — resolve sizing outside-in, never inside-out.
2. **Never bind a component to a primitive.** The chain is `component/button/bg` → `semantic/bg/accent` → `primitive/blue/500` → raw value. Bind the button straight to `blue/500` and retheming becomes a find-and-replace across every instance in every file; with the chain it is one value edit at the bottom.
3. **Cap the variant set; move the rest to properties.** `total variants = product of every axis kept as a variant`; a property costs 1, not a multiplier. A button set carrying four states, three sizes, three emphases, a leading icon and a trailing icon is 4 × 3 × 3 × 2 × 2 = 144 variants — unusable to browse and slow to edit. Above roughly 40-60 variants, convert every axis with no visual coupling — the two icon toggles here — into boolean, text, or instance-swap properties: 4 × 3 × 3 = 36 variants plus 2 boolean properties covers the same 144 combinations. State, size and emphasis stay variants because a designer compares them in the dropdown (→ Component Property Types). If the coupled axes alone still exceed the ceiling, split the set (`Button` and `Icon Button`), never delete an axis.
4. **Version breaking library changes; never restructure a main in place.** Duplicate to `Button v2`, migrate consumers file by file, then deprecate the original by prefixing its name with `_` (hides it from the assets panel without unpublishing and breaking live instances). An in-place restructure delivers the breakage to every consuming file in the same second.
5. **Name for the picker and the codebase, not the layer list.** `Button / Primary / Large` nests in the assets menu; `Frame 47` reaches engineers verbatim through Dev Mode and the REST API and ships as `frame-47`. Batch-rename with `Cmd/Ctrl + R` before marking anything Ready for dev.
6. **Build the sad paths before the prototype.** Empty, loading, error, disabled, longest-string, missing-image. A happy-path file transfers the missing states to the engineer, who invents them under deadline — and that invention is what users hit most often.
7. **Publish from a library file; consume everywhere else.** Mains living in the file where they are used get edited by accident and have no version boundary. One foundations library (variables) plus one component library per product surface; a single mega-library makes every linked file pay the load cost and every edit org-wide.
8. **Resolve the plan gate before promising a mechanism.** Mode count per collection, branching, Dev Mode seats, library analytics and the Variables REST API are all tiered (`collaboration.md`). Architecting brand × theme × density as 12 modes on a plan capped at 4 wastes the whole token structure; check `figma_plan` first.
9. **Audit responsiveness by dragging, not by looking.** Grab the frame edge and sweep the full width range the design must survive, then paste the longest realistic string. Static canvas hides clipped text, overlap, and fixed-width overflow — the three defects that reach production most often.

## Sizing Modes

| Mode | Means | Right for | Breaks when |
|---|---|---|---|
| Fill | Take the parent's available space on this axis | Cards, rows, text containers, anything responsive | The parent Hugs on the same axis: nothing to fill, size undefined |
| Hug | Shrink to fit children or text content | Buttons, chips, badges, any wrapper of text | A Fill child sits inside: the same circular dependency, from the other side |
| Fixed | The px you typed | Icon frames, avatars, sidebar rails, fixed-height app bars | Content grows past it — Figma never scrolls. With `Clip content` on it is cut off; off (the default on new frames) it spills silently over siblings |

- Responsive without breakpoints: Fill width plus `min-width` and `max-width` on the same layer. A 320/1200 clamp inside a Fill parent centers and holds across the entire viewport range — one layer instead of a breakpoint frame per size.
- `Wrap` on a horizontal Fill row replaces every nested chip-row hack, and turns gap into separate horizontal and vertical gaps.
- `Space between` with a single child left-aligns it (looks like a bug, is the rule). Two-item rows that must sit flush use `Packed` plus a Fill spacer.
- Absolute position pulls a child out of flow while keeping it parented (badges, overlays, FABs) — and it stops contributing to a Hug parent's size, which is why absolutely-positioned badges get clipped.
- Constraints (Left, Right, Scale, Center) only act inside a fixed-size frame with manual layout, or on an absolutely-positioned child. Tuning constraints inside an auto layout frame is tuning a dead control.

## Component Property Types

| Property | Cost | Right for | Trap |
|---|---|---|---|
| Variant | Multiplies the set | Axes a designer compares side by side: state, size, emphasis | Two concerns fused into one value (`Primary-Large`); value names that drift between sibling sets |
| Boolean | 1 per property | An optional sublayer: leading icon, badge, helper text | Pointing it at a layer another property also targets |
| Instance swap | 1 per property | An interchangeable child: icon set, avatar, nested card | An unbounded swap list — set preferred values |
| Text | 1 per property | Label, count, placeholder | Exposing copy that should come from a string variable |

Test for the split: an axis belongs in variants only if someone would pick it from the variant dropdown while comparing options. Everything else is a property. Property mechanics — preferred values, panel order, exposed nested instances, the `.slot` pattern — are in `components.md`.

## Token Layers

- Three layers, one direction: `primitive` (raw scale values, no meaning) → `semantic` (`bg/surface`, `text/muted`, `border/danger`) → `component` (only where a component genuinely deviates). Components bind to semantic; semantic aliases primitive.
- A semantic token with exactly one consumer that never differs across modes is noise — inline the primitive and add the token when a second consumer or a second mode appears.
- Modes retheme a whole tree in one switch: bind every fill to a mode-bound variable, then set the mode on the top frame. Child frames inherit the parent frame's mode unless explicitly overridden.
- Number variables bind to padding, gap, corner radius and stroke weight, so a spacing scale change propagates without touching layouts. Boolean variables bind to layer visibility and component boolean properties.
- Styles are not dead: gradients, effects, text and grid styles still cover what raw color and number variables do not. A style can itself reference a variable, which is the bridge during migration.

## Output Gates

Before marking frames Ready for dev or publishing a library, verify:

- Every delivered frame survives a full-width drag sweep and the longest realistic string without clipping or overlap?
- Zero layers named `Frame N`, `Group N`, `Rectangle N`, or `Ellipse N` anywhere in the delivered tree?
- Sad paths present: empty, loading, error, disabled, missing image?
- Colors, spacing and radii bound to variables — no raw hex and no typed px inside delivered components?
- Contrast checked on the bound values in every mode that ships, not only in Light?
- A breaking library change versioned rather than edited in place, with a release note naming what moved?

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/figma/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| figma_plan | starter \| professional \| organization \| enterprise | professional | Resolves every plan-gated recommendation before proposing it: mode count per collection, branching, Dev Mode seats, library analytics, Variables REST API |
| spacing_base | number (px, 2-8) | 8 | The unit every padding, gap and radius number variable snaps to; dense UI uses `spacing_base / 2` as the half-step |
| target_platforms | list (web, ios, android, desktop) | web | Selects export densities, touch-target minimums, safe-area rules and the naming used in platform specs |
| token_pipeline | native \| tokens-studio \| code-connect \| none | native | Chooses the variables-to-code route and what the handoff deliverable has to contain |
| component_naming | slash \| flat | slash | Whether component names build an assets-panel hierarchy (`Button / Primary / Large`) or stay single-segment |
| icon_grid | number (px) | 24 | Frame size for icon sets; the live area is `icon_grid − 4` and stroke weight scales from it |
| library_model | mono \| federated | federated | Whether guidance assumes one library file or a foundations library plus one per product surface |

Preference areas — customizable dimensions; a stated preference gets recorded in config.yaml and applied:

- **Tooling** — desktop app vs browser, plugin appetite (native-only vs plugin-heavy), whether the Dev Mode MCP server or the REST API is reachable from the agent
- **Conventions** — variable naming scheme, page order and cover format, layer-naming style, branch and version-naming — affects every naming recommendation
- **Platform** — density set, locale and RTL coverage, which modes ship (dark, high-contrast, compact), minimum supported viewport
- **Safety posture** — confirm before publishing a library, detaching an instance, flattening, or deleting a page; how loudly to flag a breaking change
- **Output format** — spec verbosity (annotate everything vs annotate deviations only), and whether the deliverable is a file, a written spec, or generated code
- **Work order** — tokens-first vs screens-first, when Ready for dev gets applied, whether accessibility is checked per screen or in one pass
- **Integrations** — Storybook, Jira or Linear, GitHub, Slack, Style Dictionary, Tokens Studio: which the handoff must feed
- **Restrictions** — banned plugins on enterprise files, fonts licensed for the org, compliance regimes that forbid third-party file access
- **Cadence** — library release rhythm, design review schedule, how often the unused-component and detached-instance audit runs

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Detaching an instance to get a variant that does not exist | The library link is gone for good; the missing variant never returns to the source | Add the variant or a property; detach only for one-off artwork that will never update |
| `Accept all` on a library update | A restructured main rewires overrides across every consuming file at once, silently | Review per component; open one real consuming screen before accepting a structural change |
| Duplicating frames for dark mode | Two trees drift within a sprint and every fix has to land twice | One tree, a second mode on the color collection |
| Fixed height on a text container | Localization and long strings clip with no warning on the canvas | Auto height on the text node, Hug on the parent, tested against the longest realistic string |
| Tuning constraints inside an auto layout frame | Inert except on absolutely-positioned children — the control does nothing | Fix the sizing modes; use absolute position when out-of-flow is genuinely wanted |
| `Lorem ipsum` in a mock | Hides the real length distribution; the layout breaks the day real copy arrives | Real or realistic copy plus the longest string the field permits |
| Pasting codegen output into production | px values, absolute positioning, and class names with no relationship to the codebase | Read the structure (auto layout maps to flex), rewrite against the codebase's own tokens |
| A personal access token inside a plugin bundle or a repo | Plugin bundles are readable by anyone who installs them; repo history keeps the token after deletion | Server-side proxy, scoped tokens, rotate the moment one is exposed |
| One mega-library for the whole org | Every linked file pays its load cost and every edit has org-wide blast radius | Foundations library plus one per product surface |
| Flattening an icon to "clean it up" | Loses the editable path and usually the ability to recolor | Keep the editable copy on a hidden `_source` page; flatten only decorative raster |
| Naming pages and files for the author, not the reader | The cover is the thumbnail in recents; a file nobody can identify is a file that gets duplicated | Cover page first, then `Components`, then `Screens`, `Archive` last |

## Where Experts Disagree

- **Styles vs variables-first.** New files go variables-first; teams mid-migration legitimately run both. The frontier is the coverage gap — where gradients, effects and full text-style binding are not covered by variables, a styles layer still wins. Migrating a large styles-only file is a project, not a refactor.
- **Mega-library vs federated.** Mega wins on consistency below roughly two consuming teams; past that it loses hard on load time and blast radius. The trigger to federate is a second team that needs a different release cadence, not file size.
- **Variants for every state vs separate components.** Variants win when the states share anatomy and the set stays browsable; separate components win when a "state" is really a different object with different content slots. The line is shared anatomy and set size, not taste.
- **Trust codegen vs hand-spec.** Trust generated output for layout structure and token names; never for raw px or absolutely-positioned geometry. Code Connect moves the boundary: mapped components generate real component calls, unmapped ones generate divs.
- **Design in Figma vs design in code.** Figma wins on exploration breadth and stakeholder legibility; the browser wins the moment the artifact must survive real data, real content lengths, and real device behavior. Teams that ship design-system work increasingly validate in code and keep Figma as the shared map — the split is about where the decision is verified, not about tooling loyalty.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/figma (install if the user confirms):
- `design-system` — governing the token and component architecture this file implements
- `design-tokens` — piping Figma variables through Style Dictionary into platform code
- `design-handoff` — the spec, review ritual, and async process around the Dev Mode file
- `prototyping` — when the prototype outgrows Figma (Framer, ProtoPie, Origami)
- `accessibility-audit` — running the WCAG audit against the built product

## Feedback

- If useful, star it: https://clawic.com/skills/figma
- Latest version: https://clawic.com/skills/figma

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/figma.
