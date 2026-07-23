# Prompt Engineering — Suno

## Contents

- [Structure](#structure) — the layered formula
- [Mechanics That Matter](#mechanics-that-matter) — front-load, specificity, negation, era+texture
- [Version Notes](#version-notes) — length caps and prose handling by version
- [Genre Templates](#genre-templates) — per-genre slot scaffolds
- [Voice Control](#voice-control) · [Tempo](#tempo) · [Production Style](#production-style)
- [Proven Combinations](#proven-combinations)
- [What to Avoid](#what-to-avoid)

## Structure

Build the style prompt in layers:
```
[genre] [subgenre] [mood] [tempo] [instruments] [vocals] [era/influence]
```

Example: "indie folk melancholic slow acoustic guitar soft female vocals 90s"

## Mechanics That Matter

- **Front-load.** Earlier terms steer harder in practice: genre and subgenre first, era and garnish last. If the output ignores your genre, it was probably buried.
- **Specificity beats quantity.** One precise subgenre ("shoegaze", "boom bap", "bossa nova") outsteers a pile of generic descriptors ("rock, reverb, dreamy, atmospheric"). Precise tags name a training-data cluster; generic ones average everything.
- **8-12 terms.** Every extra term past that dilutes the ones that matter.
- **One mood quadrant.** Pick energy (high/low) × valence (positive/negative) and stay inside it; "euphoric melancholic" splits the model's attention.
- **Negation fails.** "No drums" pulls drums — style terms act as attractors regardless of the words around them. Omit the term entirely, or use Suno's Exclude Styles field in custom mode.
- **Era + texture is the cheapest identity upgrade.** A generic result usually lacks a decade and a production texture: adding "90s" + "warm tape" transforms it for two tokens.

## Version Notes

- Suno >=3.5 generates 4-minute clips; older versions capped near 2 — anything longer is built by extending (→ SKILL.md Core Rule 5).
- Suno >=4.5 allows 8-minute songs, accepts style descriptions up to 1000 characters, and handles natural-language prose ("a slow-burning desert rock ballad with dusty baritone vocals") as well as comma tags; on earlier versions keep comma-separated tags.
- Current caps, model list, and pricing: check suno.com — they change between releases.

## Genre Templates

Scaffolds, not fill-in-the-blanks. Each bracketed slot lists the terms Suno actually resolves for that genre — the model has heard "boom bap" and "shoegaze" as clusters, but flattens invented compounds. Fill 8-12 terms across the slots, stay in one mood quadrant, and let a precise subgenre outsteer a pile of adjectives.

### Electronic
```
electronic [subgenre] [mood] synth [texture] [era]
Subgenres: house, techno, ambient, trance, dubstep, drum and bass, lo-fi
Textures: warm pads, crisp beats, atmospheric, glitchy, analog
Eras: 80s synth, 90s rave, modern EDM
```

### Rock
```
[subgenre] rock [energy] [guitars] [drums] [vocals] [decade]
Subgenres: alternative, indie, grunge, punk, metal, classic
Energy: driving, explosive, laid-back, aggressive
Guitars: distorted, clean, jangly, heavy riffs
```

### Pop
```
pop [mood] [tempo] [production] [vocals] [era]
Moods: upbeat, anthemic, dreamy, melancholic
Production: polished, lo-fi, synth-heavy, acoustic
```

### Hip Hop
```
hip hop [subgenre] [beat] [mood] [era]
Subgenres: boom bap, trap, lo-fi, conscious, old school
Beats: hard-hitting, laid-back, bouncy, dark
Eras: 90s golden age, 2000s, modern trap
```

### Folk/Acoustic
```
folk [subgenre] [mood] [instruments] [vocals]
Subgenres: indie folk, americana, traditional, contemporary
Instruments: acoustic guitar, banjo, mandolin, harmonica
```

### Jazz
```
jazz [subgenre] [instruments] [mood] [setting]
Subgenres: smooth, bebop, fusion, latin, modal
Settings: smoky club, late night, sophisticated
```

### Classical
```
classical [period] [ensemble] [mood] [dynamics]
Periods: baroque, romantic, modern, minimalist
Ensembles: orchestra, string quartet, solo piano, chamber
```

## Voice Control

```
soft female vocals · ethereal soprano · powerful female voice · breathy intimate female · soulful alto
deep male vocals · raspy baritone · smooth tenor · emotional male vocals
choir harmonies · gospel choir · layered vocals · duet
instrumental (and leave the lyrics box empty — stray text gets sung)
```

Combine texture + range + delivery: "raspy baritone, spoken-word verses, sung choruses" gives more control than "male vocals".

## Tempo

| Feel | Words | BPM |
|------|-------|-----|
| Fast | uptempo, driving, energetic, racing | 120+ |
| Medium | mid-tempo, steady, groovy | 90-119 |
| Slow | downtempo, ballad, laid-back | 60-89 |

Naming an explicit BPM ("128 bpm") is worth trying for dance genres; the words above are more reliable across versions.

## Production Style

```
Polished: polished clean crisp radio-ready
Raw: lo-fi raw gritty distorted vinyl warm tape hiss
Spacious: atmospheric reverb-heavy ambient expansive
Minimal: minimal stripped-down sparse
```

## Proven Combinations

```
Chill study: lo-fi hip hop chill relaxing jazzy piano beats
Epic cinematic: orchestral cinematic epic sweeping dramatic
Summer: tropical house upbeat sunny feel-good female vocals
Ballad: emotional piano ballad soft vocals heartfelt intimate
Club: EDM house energetic driving build-up heavy drop
```

When one of these lands for the user, store the exact string in `~/Clawic/data/suno/memory.md` and reuse it verbatim (→ SKILL.md Core Rule 2).

## What to Avoid

| Don't | Why | Instead |
|-------|-----|---------|
| Artist names | Moderation rejects or strips them; credits spent | Voice + era + production attributes |
| "Like [song title]" | Filtered or ignored | Describe its elements |
| Brand names | May be filtered | Generic terms |
| Negations ("no drums") | The noun still attracts | Omit, or Exclude Styles field |
| 15+ keywords | Dilutes every term | 8-12, precise over plentiful |
| Mixed mood quadrants | Splits the model | One quadrant per song |

Output still wrong after these mechanics → SKILL.md Quick Reference routes to the symptom chains before spending another run.
