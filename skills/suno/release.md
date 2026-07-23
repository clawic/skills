# Release — Downloads, Stems, Rights, Distribution

From a clip you like to a deliverable someone can use. Plan-gated features throughout: `plan_tier` in config decides which paths below are open.

## Downloads

- MP3 on all tiers; WAV on paid tiers (as of 2026 — verify at suno.com/account). `audio_format` in config sets the default.
- Download soon after generation. Via API, immediately — audio URLs expire (SKILL.md rule 7). Via browser, do not rely on page URLs surviving across sessions.
- Save to `~/Clawic/data/suno/songs/` with a stable convention: `YYYY-MM-DD-<slugged-title>-vN.mp3`. Log prompt, clip URL, and plan tier beside it in the project file — this record is the provenance answer when rights questions come later.

## Stems

- Paid feature: a vocal/instrumental split at minimum, finer multi-stem separation on newer models — the split count varies by plan and model, check the current matrix.
- Reach for stems when: video needs the music ducked under dialogue; a remix or DAW pass is planned; game audio needs adaptive layers; the vocal needs to sit louder than Suno mixed it.
- Stems are separated after the fact, not the original multitrack — dense mixes produce bleed and artifacts; sparse arrangements separate cleanly. If stems are the point, prompt "minimal" production up front.

## Remaster

- Re-renders a clip at higher fidelity on a newer model (where offered): melody and lyrics kept, mix modernized.
- Worth trying on: a keeper clip from an older model, or loudness mismatch across an extend-built song's segments.
- A remaster is another sampling roll — it can lose the nuance that made the original work. Keep the original; compare before replacing.

## Rights

As of 2026 — Suno's terms change; verify before anything ships.

| Plan at generation time | Commercial use | Ownership |
|---|---|---|
| Free | No — non-commercial with attribution | Suno retains ownership |
| Paid | Yes | Yours under the then-current terms |

- Rights attach at generation time under the plan then active. Upgrading later does not retroactively license old free-tier songs — regenerate deliverables under the paid plan.
- Purely AI-generated music is not copyrightable under US Copyright Office guidance; human-written lyrics are. Practical effect: paid output can be licensed and sold, but a similar AI melody made by someone else cannot be stopped.
- Client work: put plan tier + generation date in the project file and hand it over with the audio — that line is the license evidence the client's lawyer asks for.

## Distribution (Spotify, Apple Music, YouTube)

- Stores are reached through a distributor; AI-content policies differ per distributor and change — check the current policy of the chosen one before uploading, and disclose AI generation wherever the form asks. Nondisclosure is the takedown risk, not the AI itself.
- Keep the provenance record (prompt, date, plan, clip URL): it answers distributor questionnaires and disputes claims if a platform's matching system misfires.
- Background use in videos and podcasts needs no distributor — the paid-tier license covers it; the project-file record covers the client.

## Deliverable Gate

Before handing audio to anyone:
- Generated under the plan the use case requires?
- Format matches the destination (WAV for editors and DAWs, MP3 for direct listening)?
- Filed in `songs/` with prompt + URL + plan logged in the project file?
- For distribution: distributor's AI policy checked and disclosure made?
