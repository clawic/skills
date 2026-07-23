# Estimating — Photos, Text Logs, and Home Cooking

Estimation is the core craft: fast, honest ranges beat slow, fake precision. Error bands live in SKILL.md rule 1; anchors in SKILL.md Everyday Anchors.

## The Universal Procedure

1. Itemize foods. For photos, use scale references: standard dinner plate ~27 cm, fork ~19 cm, credit card 8.5 cm. Depth is invisible from above — for bowls and stacked food, ask for (or assume and state) a side angle.
2. Portion via hand heuristics when weight is unknown (Precision Nutrition method): palm of cooked protein = 100-120 g (~25-30 g protein); cupped hand of carbs = ~25-30 g carbs; thumb of fat = ~10 g (~90 kcal); fist of vegetables = ~1 cup.
3. Add hidden calories — the #1 undercounting source: unknown cooking method +50-100 kcal; fried +15-20%; sauce or dressing not on the side +50-100 kcal; cooking oil per SKILL.md anchors.
4. Apply context: restaurant bias per rule 1; audit the total against macros with Atwater factors (SKILL.md anchors).
5. Output the range, log the midpoint adjusted per rule 2, save recurring items to the library.

## Cost Discipline

Text first, photo only when the dish is mixed or unfamiliar. One label photo yields permanent accurate data and beats ten photo estimates. Library reuse beats everything: a confirmed repeat meal has near-label accuracy at zero effort.

## Vague-Log Defaults

Apply when the user gives a name and nothing else; per `clarify_style`, ask the one question that moves the number most, or state the assumption.

| Log says | Assume | Range |
|---|---|---|
| "pizza" | 2 chain slices | 500-700 kcal |
| "pasta" | 1.5 cups cooked + oil-based sauce | 450-600 kcal |
| "salad" | base greens + protein + dressing ON | 300-500 kcal (150 base, dressing does the rest) |
| "sandwich" | deli-size, one spread | 400-600 kcal |
| "curry and rice" | 1 cup curry + 1 cup rice, oil-cooked | 550-750 kcal |
| "smoothie" | fruit + juice base, 400 ml | 250-400 kcal |
| "burger and fries" | chain single patty + medium fries | 850-1150 kcal |
| "eggs and toast" | 2 eggs + 2 toast, butter on both | 400-550 kcal |
| "ramen" / "noodle soup" | restaurant bowl with protein | 500-800 kcal |
| "coffee" | ask if black or milk-based; black ~5, latte per anchors | — |
| Anything else | nearest analog above, widened band | state the analog used |

The question that moves the number most is almost always one of: cooking fat, dressing/sauce, drink, or portion count ("how many slices?").

## Cooked vs Raw — the 2.7× Trap

Databases carry BOTH bases; matching the wrong one is the largest single logging error (SKILL.md Traps).

| Food | Conversion | Basis check |
|---|---|---|
| Rice | dry → cooked ~3× weight | dry ~360 kcal/100 g, cooked ~130 |
| Pasta | dry → cooked ~2.2-2.5× | dry ~370 kcal/100 g, cooked ~155 |
| Oats | dry → cooked ~2.5× in water | log the dry weight, it's what the label states |
| Meat/fish | raw → cooked loses ~25% weight (water) | 100 g raw chicken breast ≈ 75 g cooked; ~120 kcal either way — same food, both entries correct |

Rule: log whatever was weighed, against the entry with the same basis. If the user weighed nothing, hand heuristics above already assume cooked.

## Homemade Recipes and Batch Cooking

- Total the raw ingredients once (labels + anchors), divide by actual servings eaten — never by the recipe's claimed "serves N".
- Save the result to the library as `Recipe name: kcal/serving, protein/serving`; future logs of that dish cost zero re-estimation.
- Ingredient error compounds mildly, not wildly: a 10-ingredient recipe with ±15% per item lands near ±5-8% on the total (errors partially cancel) — batch math is MORE accurate than photo estimates, not less.
- The two ingredients that swing home cooking: oil actually poured (measure once, remember forever) and cheese.

## Beverages

Never skip liquids — they are the gap between a "1600 kcal day" and a real 2100. Alcohol and coffee-shop drinks per SKILL.md anchors; juice ~110 kcal/250 ml; regular soda ~140 kcal/355 ml; zero/diet sodas are genuinely ~0. Milk in tea/coffee across a day adds up: 3 splashed coffees ≈ 60-100 kcal.

## When the User Pushes for One Number

Give the midpoint WITH the band: "call it 550, honestly 450-650". Never drop the band silently — rule 1 is the product, not a hedge.
