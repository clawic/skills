# Normalization — Every Token Type

The SKILL.md table covers the common cases; this is the full catalog. Principle: every non-word token gets an explicit spoken form before synthesis — engine guessing is the failure mode. Locale overrides (decimal comma, date order, 24h): `multilingual.md`.

## Numbers

| Case | Speak as | Example |
|---|---|---|
| Ordinal | word form; dates take ordinals | 3rd -> "third"; July 23 -> "July twenty-third" |
| Decimal | "point" + single digits | 3.14 -> "3 point 1 4", never "3 point fourteen" |
| Fraction | spoken fraction | 1/2 -> "half"; 3/4 -> "three quarters"; 5/16 -> "5 sixteenths" |
| Range | "to", never a dash | 10-15 -> "10 to 15"; a spoken dash becomes "minus" |
| Negative | "minus" | -4° -> "minus 4 degrees" |
| Year | digit pairs | 1999 -> "nineteen ninety-nine"; 2007 -> "two thousand seven" |
| Quantity that looks like a year | force cardinal words in the speech string | "2026 units" -> "two thousand twenty-six units", not "twenty twenty-six" |
| Leading zeros in codes | "zero", not "oh" | 0142 -> "zero one four two" — "oh" is ambiguous with the letter O |
| Roman numeral | context decides | Henry VIII -> "Henry the eighth"; Chapter IV -> "Chapter 4" |
| Precise value | 2 sig figs unless confirmable | SKILL.md rule 5 is canonical; `number_style: exact` overrides |
| Dimensions | "by" | 3×5 -> "3 by 5"; 1920x1080 -> "1920 by 1080" |
| Arithmetic | operator words | = "equals", × "times", ÷ "divided by", ^ "to the power of" |

## Money

- Cents: $5.99 -> "5 dollars 99" (natural) or "about 6 dollars" when rounding is allowed; never "5 point 9 9 dollars".
- Currency codes and symbols both become the currency word after the amount: EUR 40 and 40€ -> "40 euros".
- Amounts to be charged, transferred, or confirmed are confirmable data: exact, per SKILL.md rule 5.
- Mixed currencies in one reply: name the currency every time — "dollars" alone is ambiguous across USD, CAD, AUD.

## Dates, Times, Durations

- Prefer relative when unambiguous: "tomorrow at 3 pm" beats "July 24th at 3 pm"; add the weekday when more than 2 days out ("Friday the 31st").
- Durations: 1h30m -> "an hour and a half"; 90s -> "a minute and a half"; ISO forms (PT2H) never reach speech.
- Timezones: convert to the user's local time; name the zone only when the conversion is uncertain ("3 pm UTC").
- `time_format: 24h` speaks "14 30" — natural in most non-US locales; never force am/pm onto a 24h user.

## Contact, Codes, Identifiers

- International phone: country code is its own group — "+34 612 345 678" -> "plus 3 4, 6 1 2, 3 4 5, 6 7 8".
- Mixed alphanumerics (flight numbers, plates, order IDs): single characters, letters by name, "zero" for 0, grouped in threes or fours with pauses.
- Similar-sounding letters (B/D/P, M/N) in critical codes: switch to NATO alphabet after one failed confirmation ("B as in Bravo").
- Email: "name at domain dot com" only when short and being confirmed; otherwise send to text (SKILL.md URL row).

## Symbols

| Symbol | Speak as | Note |
|---|---|---|
| & | "and" | inside names too: "AT and T" |
| # | "number" before digits; "hash" in tags | #3 -> "number 3" |
| @ | "at" | handles: "at clawic" |
| / | "per" in units, "or" in a/b pairs | km/h -> "kilometers per hour"; "slash" only inside spoken URLs |
| ~ | "about" | ~50 -> "about 50" |
| ° | "degrees" + scale when it matters | 20°C -> "20 degrees Celsius" |
| + | "plus" | C++ -> "C plus plus" |
| * | never spoken | footnote markers drop silently |

## Structures

- Parentheticals: promote to their own sentence or delete — a spoken parenthesis is indistinguishable from the main clause.
- Quotes: "quote... end quote" only when attribution matters; otherwise just say the content.
- Footnotes and citation markers: drop; fold the source inline ("according to Chen") when it is load-bearing.
- Line breaks from wrapped source text: join before synthesis — engines treat them as sentence ends (`debug.md`, odd pauses).

## Acronym Decision Procedure

1. User lexicon entry exists → it wins over everything (`preferences.md`).
2. Lexicalized acronyms (NASA, RAM, JSON as "jay-son") → leave as is; verify once per engine.
3. Pronounceable and commonly spoken as a word in the domain → word.
4. Otherwise → letters with spaces ("F B I").
5. Mixed forms (JPEG as "jay-peg", OAuth as "oh-auth") → respell in the speech string only (`pronunciation.md`).

First-mention rule for unfamiliar acronyms: expand once ("the Internal Revenue Service — the IRS"), then the acronym alone.
