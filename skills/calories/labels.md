# Labels — Reading Packages Without Getting Played

A label is the best routinely available data source and still carries ~20% legal tolerance (SKILL.md Traps). Extract once, audit, save to library.

## Serving-Size Traps (audit these before trusting any label)

- **Per serving vs per package.** US labels often declare implausible servings — a 500 ml drink as "2 servings", a pint of ice cream as 3. First question on every label: how much of the package was actually eaten?
- **Per 100 g vs per serving.** EU/UK labels lead with per-100 g; US labels lead with per-serving. Confirm which column you are reading before multiplying.
- **"As packaged" vs "as prepared."** Pasta, rice, oats, and cereal state DRY weight; boxed mixes may state "as prepared" assuming added butter/milk. Match the basis (`estimation.md` for conversions).
- **Rounding to zero.** US rules let <5 kcal per serving declare 0. Cooking spray is the canonical abuse: "0 kcal" per quarter-second spray, ~7 kcal per actual second of spraying — a real pan coating runs 20-40 kcal. Any "0 kcal" product used in quantity deserves this check.

## The Atwater Audit

Cross-check every extracted label: protein×4 + carbs×4 + fat×9 (+ alcohol×7) should land within ~10% of the stated calories. Outside that → misread column, wrong serving basis, or a bad label; re-extract before saving. This one-line check catches most OCR and photo-extraction errors.

## Kilojoules

Australia, NZ, and much of the EU co-label in kJ. kcal = kJ ÷ 4.184 (2000 kJ ≈ 478 kcal). If `energy_unit: kJ` is set, keep all outputs in kJ and convert formulas' kcal constants once, at output time — never mix units inside one calculation.

## Sugar Alcohols and "Net Carbs"

- Sugar alcohols are not calorie-free: maltitol ~2.1 kcal/g, sorbitol ~2.6, xylitol ~2.4; erythritol ~0.2 is the real near-zero. A "sugar-free" maltitol chocolate bar carries most of the calories of the regular one.
- "Net carbs" is a marketing construct, not a labeling standard: it subtracts fiber and sugar alcohols at full value. For calorie purposes, count total carbs from the label and let the sugar-alcohol correction above handle the rest.
- Fermentable fiber contributes ~2 kcal/g; labels already include it in the calorie total in the US, while EU labels compute fiber at 2 kcal/g explicitly — no correction needed either way, just don't add fiber calories on top.

## Claims That Change Nothing

"Natural", "organic", "gluten-free", "keto-friendly", "high-protein" alter zero calorie math. The only front-of-pack words that matter are the ones that change the serving basis ("as prepared", "made with 2% milk"). Route the health-halo conversation away: calories here, food quality in `nutrition`.

## Restaurant Nutrition Pages

Chain-published data counts as label-grade (standardized recipes, legally posted in the US for 20+ location chains) — use it verbatim over any photo estimate. Custom orders break it: added cheese, extra dressing, "make it crispy" re-adds the hidden-calorie step from `estimation.md`.

## Save Everything

Every extracted label goes to the library (`memory-template.md` format): `name: kcal [per amount], protein — label, date`. The second scan of the same product should never happen.
