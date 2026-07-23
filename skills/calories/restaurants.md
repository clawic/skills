# Restaurants, Delivery, and Drinking

Eating out is where logs die: estimates are worst (rule 1's +20-30% bias), portions biggest, and shame highest. The job is a decent range and a closed day, not forensic accuracy.

## By Venue

| Venue | Play |
|---|---|
| Chain (published data) | Use their numbers verbatim (`labels.md`); custom orders re-add hidden calories |
| Independent restaurant | Estimate the homemade equivalent (`estimation.md`), then +20-30% for oil, butter, and sugar in savory dishes. Non-chain entrées average ~1200 kcal before drinks and sides (Urban, JAMA Intern Med) — when the estimate lands far below that, re-check portion size |
| Delivery/takeout | Restaurant math, plus: containers usually hold 1.5-2 home servings; log what was eaten, not the container. Leftovers saved = calories not eaten — say so, it removes guilt about "ordering too much" |
| Buffet | Count plates, not items: a loaded 27 cm plate of mixed hot food ≈ 600-900 kcal, a dessert plate ≈ 300-500 (coarse heuristic — widen the band and move on). Itemizing 15 foods is precision theater on the worst possible inputs |
| Fast casual (burrito/bowl lines) | Sum the line: base + protein + toppings; the calorie-dense picks are rice, cheese, sour cream, guac (~230 kcal a scoop) — the bowl ranges 500-1000 depending on those four |
| Café work sessions | The pastry case and the latte count; two coffee-shop visits can quietly add 600+ kcal |

## Alcohol

- Ethanol is 7 kcal/g and satiety-invisible: drinks add calories without displacing food. Base values in SKILL.md Everyday Anchors.
- Mixers dominate cocktails: spirit + diet mixer ≈ 100 kcal; margarita 250-350; cream/tiki drinks 400+. Wine and beer are self-contained and easier to log.
- Tally drinks DURING the evening (a note per round) — next-morning reconstruction reliably loses 1-2 drinks.
- Alcohol also distorts tomorrow's data: dehydration dip, then rebound water (`trend.md`). Read nothing into the scale for 2 days after a heavy night.
- "Drunk eating" is part of the night's log, neutrally, same as any meal.

## Social Strategy (when the user asks how to handle events)

- Weekly view: bank 100-200 kcal/day for 3-4 days ahead of a known event — sound arithmetic, but not for users with restrict-binge signals (SKILL.md, Where Experts Disagree). Never suggest skipping meals the day of; it reliably ends in overshoot.
- At the event: pick the anchor (the dish that matters), go light on the incidentals (bread basket, mindless refills). One plate of intention beats grazing.
- Log the day at LOW precision and close it: "roughly 2800-3400" is a complete, successful log. The failure mode is not the big day — it is the unlogged week that follows it (SKILL.md Traps, "what the hell" cascade).

## Vacations and Travel Weeks

- Default to maintenance mode: log roughly or not at all per the user's stated preference; a week at maintenance costs a cut almost nothing, but a week of guilt costs adherence a lot.
- Expect +1-2 kg on return — water, glycogen, and gut content, not fat; it clears within about a week of normal eating (`trend.md`). Say this BEFORE they step on the scale, not after.
- Resume normal logging the first full day home; no compensatory cutting — the stall protocol and measured TDEE absorb the week automatically (`calibration.md`).
