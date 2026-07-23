# Long-Form — Briefings, Documents, and Reports Aloud

Long-form is the one sanctioned breach of `speech_budget`: the user asked for length. Structure replaces brevity — audio has no headings, no scroll-back, no skim.

## Shape

- BLUF: the whole briefing's conclusion in the first two sentences — a listener who stops early still leaves informed (SKILL.md rule 2, scaled up).
- Preview before body when over 5 minutes: "Three sections: revenue, hiring, risks." The spoken table of contents is the listener's only navigation.
- Chunk by topic at 60-90 seconds each — 150-225 words at the 2.5 words-per-second base (SKILL.md rule 1). End each chunk with a check-in question unless `checkins: false` (continuous briefing mode).
- Signpost every transition: "Moving to costs." Headers do not exist in audio; transitions are the headers.
- Recap at the end: the top 3 takeaways, again. Repetition is the listener's re-read.

## Density Limits

- At most 3 figures per chunk, each with its comparison spoken alongside ("40 dollars — half of last year's price"). A naked number evaporates within a minute.
- Names: re-attach the role on first mention per chunk ("Chen, the CFO") — the listener cannot glance up to check who Chen was.
- Lists inside long-form keep the 4-item cap (SKILL.md Quick Reference); longer lists become "the top three are..., full list in text."

## Converting Documents

| Element | Spoken form |
|---|---|
| Headers | transition sentences |
| Tables | the takeaway plus one contrast; full table offered in text (SKILL.md Writing For The Ear) |
| Charts and figures | the trend in one sentence ("revenue doubles by Q3, then flattens") |
| Footnotes | drop; inline the source only when load-bearing |
| Block quotes | attribution first, then "quote... end quote" |
| Code blocks | behavior summary, never syntax (SKILL.md Quick Reference) |
| Images and captions | speak only what the surrounding text does not already say |
| Appendices | announce they exist, never read them |

Read-back requests ("read me my draft") are verbatim territory, not conversion: normalize tokens, keep the author's words (SKILL.md, Where Experts Disagree).

## Delivery

- Always produce the text artifact too: audio is the presentation, text is the record.
- Progress markers past 10 minutes: "Halfway point." Listeners budget attention against known length.
- Re-listens tolerate 1.1-1.25 rate on request (SKILL.md Prosody); first listens stay at `default_rate`.
- After an interruption, resume with a one-sentence recap of the current chunk — never restart the briefing, never resume mid-sentence.
- If the user skips or talks over chunk endings twice, that is a `checkins: false` signal (SKILL.md rule 7): confirm and store it.
