# Style Tags — Suno

Vocabulary catalog for the style field. Precise subgenre tags steer harder than broad ones — "shoegaze" names a specific sound cluster; "rock with reverb" averages everything.

## Contents

- [Genre Tags](#genre-tags) — main + subgenre clusters Suno recognizes
- [Mood Tags](#mood-tags) — one energy×valence quadrant per song
- [Tempo Tags](#tempo-tags) · [Instrument Tags](#instrument-tags) · [Vocal Tags](#vocal-tags)
- [Production Tags](#production-tags) · [Era/Decade Tags](#eradecade-tags) · [Use Case Tags](#use-case-tags)
- [Combining Tags](#combining-tags)
- [Tag Limits (canonical)](#tag-limits-canonical) — the 8-12 rule
- [Conflicting Tags to Avoid](#conflicting-tags-to-avoid)

## Genre Tags

### Main Genres
```
rock, pop, hip hop, R&B, jazz, classical, electronic, country, folk, reggae, blues, soul, funk, disco, metal, punk
```

### Electronic Subgenres
```
house, techno, trance, ambient, dubstep, drum and bass, EDM, synthwave, lo-fi beats, chillwave, future bass, hardstyle, progressive house, deep house, tech house
```

### Rock Subgenres
```
alternative rock, indie rock, grunge, punk rock, metal, hard rock, classic rock, prog rock, post-rock, shoegaze, emo, pop rock, garage rock, psychedelic rock
```

### Hip Hop Subgenres
```
trap, boom bap, lo-fi hip hop, conscious rap, old school hip hop, underground hip hop, drill, cloud rap
```

### Other Subgenres
```
indie pop, dream pop, synth-pop, art pop, chamber pop
neo-soul, contemporary R&B, quiet storm
americana, bluegrass, folk rock
latin, reggaeton, bossa nova, salsa
```

## Mood Tags

Pick one quadrant (energy × valence) and stay inside it:

```
High energy, positive: euphoric, uplifting, triumphant, celebratory, anthemic, playful
High energy, negative: aggressive, intense, dark, chaotic, urgent, relentless
Low energy, positive: peaceful, serene, soothing, gentle, meditative, dreamy
Low energy, negative: melancholic, somber, mournful, brooding, haunting, eerie
Nuanced (use alone): bittersweet, nostalgic, wistful, yearning, introspective
```

## Tempo Tags

```
Fast (120+ BPM): uptempo, driving, racing, frantic
Medium (90-119 BPM): mid-tempo, steady, groovy
Slow (60-89 BPM): downtempo, ballad, laid-back, unhurried
```

## Instrument Tags

```
Strings: guitar, acoustic guitar, electric guitar, bass guitar, violin, cello, string quartet, orchestra
Keys: piano, synth, organ, electric piano, rhodes, wurlitzer
Percussion: drums, drum machine, 808, hi-hats
Wind/brass: saxophone, trumpet, flute, brass section, horns
Electronic: synth pads, synth leads, arpeggios, sub bass, wobble bass
```

Naming 1-2 lead instruments focuses the arrangement; listing five gets a wash.

## Vocal Tags

```
Gender/group: female vocals, male vocals, duet, choir
Texture: soft, powerful, raspy, smooth, breathy, soulful, ethereal, operatic, whispered, spoken word
Range: soprano, alto, tenor, baritone, bass
Effects: reverb, echo, autotune, vocoder, harmonized
```

Best results combine texture + range: "breathy alto", "raspy baritone".

## Production Tags

```
Polished: polished, crisp, clean, radio-ready
Raw/lo-fi: lo-fi, raw, gritty, distorted, fuzzy, warm, analog, tape, vinyl
Spatial: atmospheric, spacious, reverb-heavy, wide, intimate
Texture: layered, minimal, dense, sparse
```

## Era/Decade Tags

```
60s, 70s, 80s, 90s, 2000s, 2010s
vintage, retro, modern, futuristic, old school, new wave
```

A decade tag is the single highest-leverage identity term — it implies instruments, production, and mix conventions at once.

## Use Case Tags

```
Functional: study music, workout music, focus music, sleep music, meditation
Cinematic: cinematic, film score, epic, trailer music, soundtrack
Commercial: jingle, advertisement, corporate, upbeat positive
```

## Combining Tags

```
[genre] [subgenre] [mood] [tempo] [instruments] [vocals] [production] [era]
```

**Examples:**
```
lo-fi electronic ambient chill downtempo synth pads warm analog
alternative rock energetic driving distorted guitars male vocals raw
indie pop bittersweet piano soft female vocals intimate modern
orchestral cinematic epic strings brass dramatic building
tropical house upbeat happy synth female vocals polished modern
```

## Tag Limits (canonical)

- **Optimal: 8-12 terms.** Each term past ~12 dilutes the rest; below ~5 the model fills gaps with its defaults.
- Worked check: "lo-fi electronic ambient chill downtempo synth pads warm analog" = 8 terms covering genre, mood, tempo, instrument, production — nothing wasted.
- Suno >=4.5 also accepts prose descriptions up to 1000 characters; the 8-12 rule applies to tag-style prompts.

## Conflicting Tags to Avoid

| Bad Combination | Why |
|-----------------|-----|
| happy sad | Contradictory mood |
| fast slow | Contradictory tempo |
| minimal dense | Contradictory texture |
| lo-fi polished | Contradictory production |
| aggressive peaceful | Contradictory energy |

The model doesn't error on contradictions — it averages them, and the average is generic.
