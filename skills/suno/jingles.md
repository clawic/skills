# Jingles, Intros, and Functional Audio

Briefs where the music serves another asset — an ad, a podcast, a stream, a person's birthday. Common thread: shorter structures, hard word constraints, and deliverables trimmed after generation rather than prompted to length.

## Jingles and Ads

- Target the standard spot lengths (15 s / 30 s). Suno ignores exact-duration requests — duration text becomes style words (SKILL.md Traps). Generate a short structure, then crop:
  ```
  [Hook] 2-4 lines

  [End]
  ```
- One melodic hook repeated — not verse-chorus. The lyric IS the tagline; 10-20 words total.
- Product names read as brand names to the moderation filter — even the client's own. Test one cheap run first; if the name gets stripped, sing a generic phrase and overlay the spoken name in post (`ffmpeg`/`audio` skills).
- Style: "jingle, catchy, upbeat" + a genre matching the brand's register; polished production terms, never lo-fi unless the brand is.

## Podcast and Video Intros

- Instrumental toggle ON, lyrics box empty. Describe a bed, not a song: "podcast intro, upbeat, clean, minimal, modern" + 1-2 lead instruments.
- Leave sonic space for the voice-over: "minimal, sparse" sits under speech; "epic, layered" fights it.
- Generate a full-length clip and crop the strongest 10-20 seconds — cropping a good section beats prompting for a short one.

## Background Loops (streams, games, study)

- Long-form background: the extend loop with a static style string; low variation is a feature here — "ambient, repetitive, steady, hypnotic".
- Truly seamless loops need an external cut: Suno clips fade or resolve. Crop between two same-feel bars and cross-fade the ends in a DAW or with `ffmpeg`.
- Game audio that must react to state needs stems for layering.

## Personalized Songs (birthdays, weddings, roasts)

- Personal names sing fine and pass moderation — unlike artist and brand names. Respell hard names phonetically in the lyrics ("Siobhan" → "Shiv-awn") and keep the official spelling in the title only.
- Three concrete facts about the person outperform any pile of "amazing, wonderful, special" — the same concrete-imagery rule that governs all lyrics.
- Chorus = the name plus one repeated phrase; that is what the room sings along to. Keep it identical on every repeat.

## Meditation and Sleep

- Style: low-energy positive quadrant only ("peaceful, serene, meditative"), slow-tempo words, and omit percussion terms entirely — negating them pulls them in.
- Duration via the extend loop with an identical string; melodic drift matters less here, but joins are still audible at low volume in a quiet room — cross-fade if the use is sleep.

## Seasonal and Event

- Name the occasion in the style field ("Christmas song, sleigh bells, warm", "wedding first dance, tender") — occasion words are strong style anchors that imply instrumentation.
- Traditional melodies: naming a famous recording gets filtered like any title. Hum the public-domain melody and cover it instead.
