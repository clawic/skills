# Lyrics Guide — Suno

Everything outside `[brackets]` gets sung. That single fact drives most of this file.

## Contents

- [Structure Tags](#structure-tags) — recognized section markers
- [Delivery Control (the invisible knobs)](#delivery-control-the-invisible-knobs) — parentheses, caps, vocal cues, syllable placement
- [Structures That Work](#structures-that-work) — pop/rock, verse-heavy, EDM
- [Word Budget](#word-budget) — 150-300 words per 2-3 min
- [Endings](#endings) · [Instrumental Sections](#instrumental-sections)
- [Emotional Arc](#emotional-arc) · [Writing Judgment Calls](#writing-judgment-calls)

## Structure Tags

Recognized markers — bracketed, each on its own line:

```
[Intro] [Verse] [Verse 1] [Pre-Chorus] [Chorus] [Post-Chorus]
[Hook] [Bridge] [Break] [Instrumental] [Guitar Solo]
[Build] [Drop] [Outro] [Fade Out] [End]
```

Tags steer, they don't command: the model follows them most of the time. If a tag gets ignored, regenerate or restructure — stacking more tags at the same spot makes output worse, not better.

## Delivery Control (the invisible knobs)

- **Parentheses = backing vocals / ad-libs**: `(ooh)`, `(let's go)`, `(rise up!)` — sung under or after the main line. This is how call-and-response choruses are written.
- **ALL CAPS tends toward belted/shouted delivery**; lowercase stays soft. A tendency, not a switch — pair it with a vocal cue in the tag.
- **Vocal direction inside the tag**: `[Verse - whispered]`, `[Chorus - powerful]`, `[Bridge - spoken word]`, `[Outro - fading]`.
- **Punctuation phrases the vocal**: commas insert breaths, ellipses stretch a line... exclamation marks push energy.
- **Hyphenate to place syllables** when the model rushes or mangles a word: "to-night", "for-ev-er". Names it mispronounces get phonetic respelling in the lyrics ("Siobhan" → "Shiv-awn") — official spelling stays in the title.
- **Repeated exact text = repeated melody.** Identical `[Chorus]` text on every repeat is how you get one chorus melody instead of three different ones.
- **Language**: write lyrics in the target language and also name it in the style field ("Spanish vocals", "Japanese city pop"); untagged non-English lyrics get inconsistent pronunciation.

## Structures That Work

### Pop/Rock Standard
```
[Verse 1] 4-6 lines
[Chorus] 2-4 lines (the hook)
[Verse 2] 4-6 lines
[Chorus] identical text
[Bridge] 2-4 lines (contrast: new chord feel, new angle)
[Chorus] identical text
[Outro] 1-2 lines
[End]
```

### Verse-Heavy (Folk/Hip Hop)
```
[Verse 1] 8-12 lines (storytelling)
[Chorus] 2-4 lines
[Verse 2] 8-12 lines
[Chorus]
[Verse 3] 8-12 lines
[Chorus]
[End]
```

### EDM/Dance
```
[Intro] 2 lines or [Instrumental]
[Verse] 4 lines
[Build] 2 lines
[Drop] 2 lines or [Instrumental]
[Verse 2] 4 lines
[Build]
[Drop]
[Outro]
[End]
```

## Word Budget

| Section | Lines | Words |
|---------|-------|-------|
| Verse | 4-8 | 30-60 |
| Chorus | 2-4 | 15-30 |
| Bridge | 2-4 | 15-30 |
| Outro | 1-2 | 5-15 |

**Total: 150-300 words for a 2-3 minute track.** Overshoot and the vocal turns rushed and half-spoken; undershoot and Suno pads with repeats and instrumentals (fine for EDM, odd for ballads). Keep paired lines within a couple of syllables of each other when you want the melody to repeat cleanly:

```
Walking down the street (5)
Looking for the beat (5)     ← same melody slot

Walking down the street (5)
I am looking everywhere for something to complete (12)   ← forces a new melody
```

## Endings

Close every standalone song with `[End]` as the final tag — clips without an ending cue stop mid-phrase (→ SKILL.md Core Rule 4). Options before it:

```
[Outro]
last thematic lines

[End]
```
or, for instrumental fades: `[Fade Out]` then `[End]`. A final chorus marked `[Chorus - big finish]` before the outro lands the triumphant ending.

## Instrumental Sections

Parenthetical text inside an instrumental tag describes what plays, not what is sung:

```
[Instrumental]
(melancholic guitar solo)

[Break]
(drum break)

[Drop]
(heavy bass drop)
```

## Emotional Arc

Map dynamics onto sections so the song moves somewhere:

```
[Verse 1] establish, calm → [Chorus] release → [Verse 2] complicate, building
→ [Chorus] bigger → [Bridge] twist or peak → [Final Chorus] resolution
```

For narrative songs: setup ([Verse 1]) → conflict ([Verse 2]) → theme ([Chorus]) → resolution ([Verse 3]) → reflection ([Outro]).

## Writing Judgment Calls

- **Don't over-rhyme.** Near-rhyme reads modern; perfect AABB on every line reads nursery-rhyme. Suno handles slant rhyme fine.
- Concrete imagery over abstractions — "empty rooms and photographs" sings better than "the sadness of loss".
- Vary line lengths across sections (not within melody pairs) so the song breathes.
- One metaphor system per song; mixed metaphors become word salad at singing speed.
- Repetition is a feature: repeating a phrase 3-4 times in a chorus is what makes it a hook, not lazy writing.
