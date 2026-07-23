# Browser Automation — Suno

## When to Use

- No API key configured, browser tool available
- Testing prompts before spending API credits
- Features hosted APIs don't expose: extend from timestamp, crop, personas, stems
- User wants to listen and pick before downloading

Use whatever browser automation tool the agent has (navigate, snapshot, type, click). Steps below are tool-agnostic.

## Login First

Suno requires login to generate. If the create page redirects to login, pause automation and ask the user to complete login manually in the browser — never handle their credentials.

## Generate: Simple Mode

1. Navigate to `https://suno.com/create`
2. Snapshot the page; identify the description input
3. Type the style prompt
4. Toggle "Instrumental" if no vocals wanted
5. Click Create; generation takes 30-90 seconds and returns 2 clips
6. Listen/evaluate both — they are two rolls of the same prompt

## Generate: Custom Mode

Custom mode separates lyrics from style — use it whenever the user has words or structure in mind:

1. Enable the Custom toggle
2. Lyrics box: structured lyrics with `[Verse]`/`[Chorus]` tags, `[End]` last
3. Style box: 8-12 comma-separated terms
4. Exclude Styles (if present): genres/elements to push away — this is the only working negation
5. Advanced sliders (versions that expose them): raise Style Influence when output ignores the prompt; Weirdness trades coherence for novelty. Defaults first; move one slider at a time
6. Optional title, then Create

## Extend, Crop, Covers, Personas

Full workflows with segment planning live in `extend.md` (long tracks, crop, Get Whole Song) and `covers.md` (covers, personas, uploads). UI mechanics: Extend, Crop, and Cover sit on the clip's row/three-dot menu; when extending, set the extension-point slider to the chosen timestamp before confirming — the default is the clip's end, which is usually the degraded part.

## Downloading

- From library: three-dot menu on the song → Download → MP3 (paid tiers also expose WAV and stems where available)
- Download soon after generation when automating; don't rely on page URLs staying valid across sessions

## Credits and Limits (as of 2025 — verify at suno.com/account)

- Free tier: 50 credits/day; one generation costs 10 credits and returns 2 clips → 5 runs (10 clips) per day
- Worked example: an extend-built 4-minute song needing 3 extend runs plus 2 initial attempts = 5 runs = a full free day. Budget before starting, or confirm a paid plan
- Commercial rights require a paid plan at generation time

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Create button disabled | Not logged in or out of credits | Check login, then credit balance |
| Generation stuck >5 min | Server busy | Reload page; the task usually completed server-side |
| Download fails | Stale page state | Reload library, retry from three-dot menu |
| Both clips off-genre | Prompt buried the genre | Front-load genre terms, regenerate |
| Words missing from the sung lyrics | Moderation stripped artist/brand names; credits still spent | Pre-convert names to attributes (SKILL.md rule 6) |
| Output wrong in any other way | See the symptom chains | SKILL.md Quick Reference routes to the fix before spending another run |
