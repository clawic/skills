---
name: Recipes
slug: recipe
version: 1.0.2
description: Captures, standardizes, scales, and files recipes into a personal collection that stays searchable and cookable. Use when the user pastes a recipe URL, photo, screenshot, video, voice note, or handwritten card to save; asks to halve, double, or triple a dish, or fit it to a different pan or tin; converts cups to grams, ounces to millilitres, Fahrenheit to Celsius, or gas marks, or adjusts for a fan oven or altitude; asks what to cook from what is saved, or searches by ingredient, tag, time, or rating; builds a week from the collection with one combined shopping list; swaps a missing, disliked, or off-limits ingredient; rewrites a dish for a pressure cooker, air fryer, or slow cooker; works out cost per serving; writes or tests an original recipe; preserves a family recipe; or moves a collection between apps. Not for stove-side rescue (`cooking`), general cooking help (`chef`), household meal systems (`meal-planner`), calories and macros (`calories`), or pantry lists (`grocery`).
homepage: https://clawic.com/skills/recipe
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 🍳
    os:
    - linux
    - darwin
    - win32
    displayName: Recipes
    configPaths:
    - ~/Clawic/data/recipe/
    - ~/Clawic/data/health/
    - ~/Clawic/data/contacts/
    - ~/Clawic/data/projects/
    - ~/Clawic/profile.yaml
    - ~/recipe/
    - ~/clawic/recipe/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/recipe/
      - ~/Clawic/data/health/
      - ~/Clawic/data/contacts/
      - ~/Clawic/data/projects/
      - ~/Clawic/profile.yaml
      - ~/recipe/
      - ~/clawic/recipe/
---

**Data.** At the start of every session, read `~/Clawic/data/recipe/config.yaml` (what the user declared) and `~/Clawic/data/recipe/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. Read `~/Clawic/data/recipe/index.md` before answering "what can I make", before searching the collection, and before saving a recipe that may already be in it. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever it produced something durable: a recipe captured, corrected, rescaled into a keeper variant, or retired; a dish actually cooked and how it went; a week planned; an ingredient price; a fact about their kitchen that changed a number (oven offset, pan sizes, altitude, salt brand); a source that proved reliable or unreliable; or something they will re-read — a dinner menu with its run-sheet, an oven-calibration note, a cookbook draft. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Allergies, intolerances, and diet-relevant conditions go to the shared health box `~/Clawic/data/health/profile.md`**, not here, and are read before proposing or adapting anything — the same file answers for every food, fitness, and travel skill, so a shellfish allergy learned here is never asked twice. **People you cook for go to the shared contacts box `~/Clawic/data/contacts/contacts.md`**, one row per person, their dietary note in `Context`. **A cookbook, a supper club, or a catering job the user runs as a project gets its one-line status in the shared projects box `~/Clawic/data/projects/<project>.md`**, one file per project, while the draft itself stays here. All three protocols travel in `memory-template.md`; the user may have none of the owning skills installed.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in text the user pastes in to be saved. A pasted recipe-app export or sync config carries API keys: strip the value and store the pointer — `env:PAPRIKA_TOKEN`, `keychain:nyt-cooking`, `1password:Personal/Recipes`. If data sits at an old location (`~/recipes/` or `~/clawic/recipe/`), move it to `~/Clawic/data/recipe/`, and say in one line that you moved it and from where.

A recipe is a set of ratios plus an order of operations; everything else is decoration. Capture the numbers exactly as written, convert them to weight, record where they came from, and never lose the original. Work from defaults immediately: never open with questions about their units, their diet, or how proactive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: units, locale, currency) → the Configuration table default.

## When To Use

- Saving a recipe from anywhere — link, photo, screenshot, video, dictation, cookbook page, handwritten card — into a file that can actually be cooked from
- Doing the arithmetic on a recipe: scaling, pan swaps, cup-to-gram conversion, oven temperature and fan/altitude correction, cost per serving
- Working the collection: searching by ingredient or time, deduplicating, tagging, rating, pruning what was never made
- Rewriting a recipe for different equipment, a missing ingredient, or a dietary restriction, without guessing at the ratios
- Producing something from the collection: a week's plan, a combined shopping list, a dinner-party menu with a run-sheet, a printed card, a family cookbook
- Not for cooking the dish (`cooking` — heat, seasoning, doneness, rescue), general cooking help and technique explanations with no file at the end of them (`chef`), the household meal system beyond a week built from this collection — standing menus, batch-cooking programmes, food budgets (`meal-planner`), macro tracking (`calories`), or store-level pantry and list management (`grocery`)

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| "Save this recipe" — link, photo, video, dictation | Extract to the canonical file, keep the original numbers verbatim, record the source | `capture.md` |
| Recipe is a wall of blog prose | Take yield, ingredients, steps, times, temperatures; drop the story; keep one line of the headnote if it states a technique | `capture.md` |
| "Is this recipe any good?" / it looks off before cooking | Run the ratio sanity checks — hydration, salt %, leavener, fat, time-vs-temperature | `vetting.md` |
| Writing or fixing the recipe file itself | Frontmatter schema, ingredient-line grammar, step grammar, yield statement | `format.md` |
| Cups to grams, °F to °C, gas marks, fan oven, altitude | Density per ingredient, never a universal factor; fan −20 °C; boiling point drops ~1 °C per 300 m | `conversion.md` |
| Halve, double, triple, or fit a different tin | factor = target ÷ base, then fix the non-linear terms (→ What Does Not Scale) | `scaling.md` |
| Missing, disliked, or off-limits ingredient | Substitution ratios by function (bind, leaven, brown, thicken, acidify), marked untested | `substitutions.md` |
| Vegan, gluten-free, dairy-free, low-sodium version of a saved recipe | Convert by function, then re-check the leavening and the liquid | `substitutions.md` |
| Pressure cooker, air fryer, slow cooker, sous vide, bread machine, oven-to-hob | Time and liquid mapping per device, and what does not transfer at all | `equipment.md` |
| "I made it, here's what I changed" | Make-log row, rating, and the change promoted into the recipe's Variations | `testing.md` |
| "What can I make?" / find by ingredient, tag, or time | Query the index, not the folder; tag vocabulary and dedup rules | `library.md` |
| Plan a week and give me one shopping list | Pick for overlap, aggregate lines to one unit per ingredient, round to package sizes | `planning.md` |
| "What does this cost per serving?" | Unit price × quantity ÷ edible yield, summed, divided by servings | `costing.md` |
| Writing an original recipe for other people | Test rounds, yield accuracy, ingredient ordering, headnote, what a tester needs | `authoring.md` |
| Grandma's card, a relative dictating from memory, a fading photocopy | Preserve the original artifact first, then build the tested version beside it | `preservation.md` |
| Import from or export to an app; back up the collection | Field mapping, JSON-LD, dedup on merge, print cards | `migration.md` |
| Anything else recipe-shaped | Answer from the collection first — cite the saved file — then from general knowledge, and say which | — |

Coverage map: `capture.md` getting it in · `vetting.md` will it work · `format.md` the file · `conversion.md` units and temperatures · `scaling.md` batch and pan math · `substitutions.md` swaps by function · `equipment.md` device rewrites · `testing.md` cooking it and recording it · `library.md` search, tags, dedup, pruning · `planning.md` weeks and shopping lists · `costing.md` cost per serving · `authoring.md` writing for others · `preservation.md` heirloom recipes · `migration.md` import, export, backup.

## Core Rules

1. **One recipe, one file, from the first one.** `~/Clawic/data/recipe/recipes/<kebab-title>.md`, plus its row in `index.md`, written in the same turn. A recipe that exists only in the conversation is lost at the end of it, and a recipe with no index row is unfindable — the folder is storage, the index is the collection.
2. **Capture verbatim, normalize second.** Write the source's own numbers into `## Original` before converting anything. Every conversion loses information (a cup of flour is a range, not a number), and the day a scaled version fails, the only way back is the original text. This is not optional for handwritten and family recipes (`preservation.md`).
3. **Weight is the unit of record for anything baked or scaled.** Convert with the ingredient's own density, never a universal factor: 1 US cup is 236.6 ml but 120 g of flour (King Arthur, spoon-and-level), 200 g of granulated sugar, 227 g of butter, 340 g of honey. Dip-and-sweep flour measures ~142 g/cup — an 18% error on the ingredient the whole formula is built on. Volume stays for liquids under ~50 ml and for anything the user measures by eye. Governed by `weight_over_volume`.
4. **Scale by ratio, then fix the terms that are not linear.** `factor = target_servings ÷ base_servings`. Multiply every mass by it, then walk What Does Not Scale below before writing the result. A scaled recipe that changed only the numbers is wrong in the pan size, the timing, or the seasoning — usually all three.
5. **Time and temperature belong to thickness, not to mass.** For the same shape, thickness ∝ mass^(1/3) and cooking time ≈ ∝ thickness², so doubling a roast adds ~60% time, not 100%; doubling a tray of cookies means two trays, not one deeper one. Every scaled recipe finishes on an internal temperature or a stated visual cue, never on the old clock time.
6. **Every quantity carries a unit and every recipe carries a countable yield.** "4 servings" is unverifiable; "12 muffins" or "4 servings × ~350 g" is. Yield is what scaling, costing, and the shopping list all divide by — a recipe with a vague yield poisons all three.
7. **Attribution travels with the recipe.** URL with capture date, or book plus edition and page, or person plus year. Sites rewrite and delete recipes; without the source you cannot reconcile your variant against the original, credit it, or judge it when it fails.
8. **A substitution or a scale is untested until the make log says otherwise.** Mark it `untested` in the recipe file. `testing.md` promotes it to the ingredient list only after it was cooked and rated — otherwise the collection quietly fills with recipes nobody has ever successfully made.
9. **Nothing enters or leaves the collection silently.** Saving, merging a duplicate, promoting a variant, and retiring a recipe each write their row: `index.md` for the entry, `made/<year>.md` for the cook, `memory.md` `## Boxes` for any new file. Retirement means the row moves to `## Retired` in `index.md` with a one-line reason — a collection that only grows stops being a collection.

## Measures That Change The Dish

Same word, different quantity. Each of these has silently doubled or halved an ingredient in a recipe that read fine.

| Measure | The trap | Use |
|---|---|---|
| Cup | US 236.6 ml · metric (AU/NZ) 250 ml · Japan 200 ml · UK legacy 284 ml | Grams; if the source is a cup, record which cup in `## Original` |
| Tablespoon | 15 ml almost everywhere, **20 ml in Australia** — 4 teaspoons, not 3 | Millilitres or grams; an AU tablespoon of salt is a third more salt |
| Kosher salt | Diamond Crystal ~2.8 g/tsp · Morton ~4.8 g/tsp · fine table salt ~6 g/tsp | Grams always. A recipe written in Diamond Crystal, measured in table salt by volume, is ~2× the salt |
| Egg | US large 50 g out of shell (white 30 g, yolk 20 g); EU L is 63-73 g **in shell** — a US large is closer to an EU medium | Grams for anything baked; whisk and weigh to get a half egg |
| Stick of butter | US only: 113 g (4 oz, 8 tbsp) | Grams |
| Flour, 1 cup | 120 g spoon-and-level, ~142 g dip-and-sweep, more again if packed | Grams; if the source is US volume, assume 120 g and say so |
| Oven °C/°F | Fan/convection runs hotter than the dial: reduce the setting by 20 °C (35 °F) or the time by ~25%, not both | State conventional or fan in every temperature, driven by `oven_type` |
| Gas mark | GM4 = 180 °C, GM1 = 140 °C, GM9 = 240 °C — the steps are uneven (10 or 20 °C), so read the mark off the table in `conversion.md` and never interpolate | Convert to the user's `temperature_scale` and keep the mark in `## Original` |
| "1 can" / "1 packet" | Can and packet sizes differ by country and by decade | Record the gram weight on the tin, not the count |
| "Medium onion" | ~150-200 g raw; recipes disagree by a factor of two | Grams, with the count in parentheses |

## What Does Not Scale Linearly

Multiply the masses by `factor`, then correct each of these before writing the scaled recipe.

| Term | Behavior | Rule |
|---|---|---|
| Pan or tin | Area, not volume, tracks the batch | Round: area = π(d÷2)². 20 cm → 314 cm², 23 cm → 415 cm² = ×1.32. Keep the depth, scale the area by `factor`; a doubled batter in the same tin bakes raw in the middle |
| Oven and roasting time | Tracks thickness² | Double the mass of the same shape ≈ ×1.6 time (Rule 5); finish on internal temperature |
| Salt and seasoning in a pot dish | Linear, but error is one-directional | Scale it, then at `factor` ≥ 2 add 80% of the scaled amount and correct at the end. You can add salt; you cannot remove it |
| Chili, garlic, strong spice, alcohol | Perceived intensity grows faster than the dose | At `factor` ≥ 2, start at 75% of the scaled amount; also gated by `spice_level` |
| Chemical leavener | Linear against flour weight, and band-limited | Hold 1-1¼ tsp baking powder (4-5 g) per 120 g flour. If the scaled amount leaves that band, the original was already out of it — the one exception is a recipe built on self-raising flour, which carries ~6 g per 120 g by specification (`substitutions.md`) |
| Yeast | Scales with the dough, not with the clock | Keep the baker's percentage: 0.5-1% instant yeast on flour for same-day, 0.1-0.3% for an overnight cold ferment. Changing the schedule changes the yeast; changing the batch does not |
| Salt in bread dough | Fixed percentage | 1.8-2.2% of flour weight, always |
| Reduction and evaporation | Rate ∝ exposed surface area | Doubling a sauce in the same pan doubles the reduction time; use a wider pan to keep the time, or accept the longer cook |
| Frying oil, blanching water | Governed by thermal mass, not by recipe factor | Batch so the oil recovers to temperature within ~1 minute of the food going in; more food per batch, not more time |
| Eggs and other countable units | Land on fractions | Weigh: 25 g of whisked egg is half a US large egg. Never round a half egg up in a batter |

## Output Gates

Before delivering a recipe, a scaled version, a conversion, or a plan:

- Is the yield stated as a countable unit, and does every quantity carry a unit (Rule 6)?
- Was this scaled? Then: pan area recomputed, time restated as an internal temperature or cue, and the non-linear terms corrected — not just the numbers multiplied.
- Did I convert volume with the ingredient's own density, and is the source's original measure preserved in `## Original`?
- Does anything here conflict with `~/Clawic/data/health/profile.md` (allergies, conditions) or the `diet` variable? An allergen is never substituted silently — name it, then offer the swap.
- Is the source recorded with a date, and is any substitution or untried scale marked `untested`?
- Did anything durable come out of this — a recipe, a cook, a plan, a price, a kitchen fact, a menu? Then it is written to its box in `memory-template.md`, with its `## Boxes` or `index.md` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/recipe/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| units | metric \| us-customary \| uk-imperial | metric | The system every captured recipe is converted into; `## Original` always keeps the source's own system (`conversion.md`) |
| weight_over_volume | bool | true | Whether ingredient lines lead with grams and keep volume in parentheses, or the reverse (Rule 3) |
| temperature_scale | celsius \| fahrenheit | celsius | Every oven temperature and doneness target; gas marks are always converted into it |
| oven_type | conventional \| fan \| gas \| none | conventional | Applies the fan offset (−20 °C) or the gas-mark table to every baked recipe (`equipment.md`) |
| altitude_m | number (metres, 0-4000) | 0 | Above 1000 m, triggers the leavener, liquid, sugar and boiling-point corrections in `conversion.md` |
| default_servings | number (1-20) | 2 | The yield captured recipes are offered scaled to, and the divisor in `costing.md` |
| diet | list (vegan, vegetarian, gluten-free, halal, kosher, low-sodium, …) | none | Filters suggestions and pre-applies the conversions in `substitutions.md`. Chosen patterns only — allergies live in the shared health box, never here |
| spice_level | mild \| medium \| hot | medium | Scales chili and pepper quantities when adapting or scaling a recipe |
| currency | text (ISO code) | from `profile.yaml`, else USD | The currency every price and cost-per-serving carries in its value (`costing.md`) |
| index_grouping | course \| cuisine \| none | course | How `index.md` is ordered, and the axis the collection splits on when it outgrows one file |
| pantry_staples | list | salt, pepper, cooking oil, water | Ingredients excluded from generated shopping lists (`planning.md`) |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — whether recipes are also emitted as JSON-LD or a print card, which app the collection syncs with, plain files versus an export target (`migration.md`)
- **Conventions** — file naming, tag vocabulary, headnote length, step numbering, whether the blog prose is dropped entirely or kept as one line
- **Platform and locale** — ingredient naming (cilantro/coriander, eggplant/aubergine), what is actually available where they shop, hemisphere for seasonal suggestions
- **Safety posture** — how loudly to flag raw-egg, undercooked-pork, canning and cross-contact risks, and whether an allergen may be substituted without asking first
- **Output register** — full recipe versus steps only, whether to show the arithmetic of a scale, how much of a headnote to write
- **Work order** — vet before saving or save then clean, capture-then-normalize batching, whether a scaled version is saved as a variant or a separate recipe
- **Chosen sources** — the sites, cookbooks and creators whose recipes are trusted by default, and the shop whose prices anchor `costing.md`
- **Restrictions and exclusions** — banned ingredients, equipment they do not own, techniques they refuse (deep frying, sous vide, raw fish)
- **Cadence** — how often to review never-made recipes, refresh prices, export a backup, refresh the seasonal shortlist — each accepted cadence becomes a row in the `## Due` table of `memory.md`

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Saving the link instead of the recipe | Sites paywall, rewrite, and delete; the bookmark survives, the recipe does not | Extract to a file, keep the URL as attribution with its capture date (`capture.md`) |
| Converting cups with one universal factor | A cup is a volume; every ingredient has a different density | Per-ingredient density table (`conversion.md`) |
| Doubling a cake and using the same tin | Depth doubles, so the outside sets before the centre cooks | Scale the pan area, keep the depth (What Does Not Scale) |
| Multiplying the oven time along with the recipe | Time follows thickness, not mass (Rule 5) | Restate the finish as an internal temperature or a visual cue |
| Trusting "1 tsp salt" across salt brands | Diamond Crystal to table salt by volume is roughly a doubling | Record salt in grams and name the brand in `## Original` |
| Rewriting grandma's card into modern units and discarding the card | The odd measure ("a coffee cup of flour") is the only calibration that exists | Photograph the original, keep it verbatim, build the tested version beside it (`preservation.md`) |
| A collection of 400 recipes and 9 ever cooked | Hoarding feels like cooking; the index gets useless faster than it gets big | `Made` count in the index; review never-made recipes on a cadence, retire with a reason (`library.md`) |
| Substituting an allergen without saying so | The user needs to know their dish changed, and a silent swap in a saved file becomes an invisible risk later | Name the allergen, name the swap, mark it `untested`, check the shared health box first |
| One shopping list per recipe | The overlap between recipes is where the saving and the waste both live | Aggregate to one unit per ingredient, then round to package sizes (`planning.md`) |
| Pressure-cooker time taken from the oven version | Pressure changes the mechanism, not just the speed, and the liquid minimum is a hard floor | Device mapping table (`equipment.md`) |
| Recording a make as "good" | Next month it answers nothing | Rating plus the one change you made plus the one thing to fix (`testing.md`) |
| Editing the recipe file mid-cook from memory | The change that mattered is forgotten by the time the plates are cleared | Make-log row first (`made/<year>.md`), promote to the recipe when it repeats |

## Where Experts Disagree

- **Volume versus weight.** Baking is settled — grams, because a cup of flour varies ~18% by hand. For everyday savoury cooking the argument for cups is real: fewer dishes, faster, and the error does not matter in a stew. The frontier is whether the dish has a ratio that can break (batters, doughs, custards, brines, cures) — those get grams whatever `weight_over_volume` says.
- **Keeping the blog prose.** Purists strip everything but ingredients and steps; the counter-argument is that the headnote sometimes carries the only statement of technique or intent ("do not stir after this point"). Default: drop the story, keep any sentence that changes what you do.
- **Baker's percentage for everything.** Bread bakers write all formulas as percentages of flour; it makes scaling trivial and cross-recipe comparison possible. Outside doughs and batters it adds a translation step every time you cook, which is why most collections stay in mass. Use percentages where flour is the backbone, mass elsewhere.
- **How faithful a captured recipe should be.** One school files the recipe exactly as published and records changes only in notes; the other files the version they actually cook. Both work; mixing them does not. The Original/Ingredients split in `format.md` is what lets you hold both without ambiguity.
- **Star ratings versus make counts.** Ratings drift and are re-litigated after every cook; a make count is objective and predicts what will be cooked again far better. Keep both, sort by makes.

## Security & Privacy

**Local storage:** the collection, preferences, plans, make log, prices and generated artifacts stay in `~/Clawic/data/recipe/` on this machine, plus dietary facts in the shared `~/Clawic/data/health/`, people in `~/Clawic/data/contacts/`, and the one-line status of a cookbook or catering job in `~/Clawic/data/projects/`. Recipe text, sources and quantities only.

**Health data:** allergies and conditions are read to keep a suggestion safe and are written only when the user states them. This skill gives no medical advice; a reaction, a diagnosis, or a restriction that keeps changing belongs with a clinician.

**Capture:** fetching a recipe URL reads a public page. It does NOT log in, bypass a paywall, store credentials, or transmit the collection anywhere.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/recipe (install if the user confirms):
- `cooking` — heat, seasoning, doneness and rescue once the recipe is in front of the stove
- `meal-planner` — longer-horizon menus, batch cooking and household logistics
- `grocery` — pantry inventory and store-level list management
- `calories` — calories, macros and targets for what gets cooked
- `nutrition` — micronutrient gaps and dietary-pattern coverage

## Feedback

- If useful, star it: https://clawic.com/skills/recipe
- Latest version: https://clawic.com/skills/recipe

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/recipe.
