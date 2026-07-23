# Debugging — Symptom to Cause

Every reroll costs credits, so diagnose before regenerating. Each chain is ordered by probability; each step is a check, not a guess.

## The Universal First Three

1. **Which box got which text?** Style words in the lyrics box (or lyrics in the style box) explains most weird output — SKILL.md rule 1.
2. **Simple or custom mode?** Simple mode reinterprets your whole text as a description; custom mode is the only place lyrics, style, and Exclude Styles are separate controls.
3. **Which model version?** Length caps, prose handling, and tag adherence differ across versions — check the model selector before blaming the prompt.

## Sounds Generic

1. No era or texture terms → add a decade + production texture ("90s" + "warm tape") — the cheapest identity upgrade, two terms.
2. Generic tags averaging everything → swap the pile for one precise subgenre ("shoegaze", not "rock, reverb, dreamy").
3. More than 12 terms diluting each other → cut to 8-12.
4. Contradictory tags averaging to mush (lo-fi + polished) → one mood quadrant, one production family.
5. Style Influence slider low (versions that expose sliders) → raise it; move one slider at a time.

## Off-Genre

1. Genre buried late in the prompt → front-load it; earlier terms steer harder.
2. Lyrics pull against the style — dark imagery drags a sunny genre. The lyrics steer too; align them or accept the blend.
3. Ambiguous single-word tag ("trap", "drill", "garage") → use the fuller form ("trap hip hop", "UK garage").
4. Both clips off-genre two runs straight → the term combination names no real cluster; change the subgenre itself, not more adjectives around it.

## Vocals Garbled or Rushed

1. Over the word budget (150-300 words per 2-3 minute track) → cut lines, not syllables.
2. One line crams syllables against its melody pair → match paired lines within a couple of syllables.
3. Model mangles a word or name → hyphenate to place syllables ("for-ev-er") or respell phonetically ("Siobhan" → "Shiv-awn").
4. Non-English lyrics without a language tag → name the language in the style field ("Spanish vocals").

## Melody Misbehaves

- Different melody on every chorus → chorus text differs between repeats; paste identical text.
- Paired lines refuse to share a melody → syllable counts differ; even them out.
- A structure tag gets skipped (`[Guitar Solo]` ignored) → tags steer, they don't command; regenerate or restructure — stacking more tags at the same spot makes output worse.

## Endings and Length

- Stops mid-phrase → no `[End]` cue (SKILL.md rule 4).
- Song shorter than wanted → version clip cap or thin lyrics padded out; build longer by extending (`extend.md`).
- Outro rambles → put `[End]` directly after a 1-2 line `[Outro]`; for instrumentals `[Fade Out]` then `[End]`.

## Instrumental Problems

- Vocals appear on an instrumental → both conditions must hold: instrumental toggle ON and lyrics box empty.
- Wordless vocalise creeps in → remove vocal-adjacent style terms (choir, vocals, singer, ballad); omit rather than negate.

## Moderation Rejections

- Rejected or silently altered → artist names, brand names, recognizable copyrighted lyric lines, or explicit terms. Credits are charged either way — pre-check every proper noun before submitting.
- Personal names (birthday songs, dedications) pass; artist and brand names do not.
- Repeated rejections with no obvious cause → submit lyrics with a bland style, then the style with bland lyrics, to isolate which half trips the filter.

## Extend Joins Sound Wrong

- Audible seam → style string changed between segments; keep it byte-identical.
- Tempo or key jump at the join → extension point sits mid-phrase; pick a boundary at the end of a musical phrase.
- Voice changes character after the join → you extended from a sibling roll, not the clip lineage you chose; extend from the same clip every time, or lock the voice with a persona.

## When Truly Stuck

Strip to a minimal simple-mode prompt: one genre + one mood. Confirm that lands, then re-add lyrics, then tags, one element per run — the element that breaks it names what to fix next. Each step costs a run: budget credits before starting.
