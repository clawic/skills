# Long Tracks — Extend, Crop, Stitch

When a song needs more than one clip delivers: past the version's clip cap, fixing a section that drifted, or salvaging the good half of a roll. Version caps: Suno >=3.5 generates 4-minute clips; >=4.5 one-shots up to 8 minutes but drifts in melody and mix past ~4 (SKILL.md Where Experts Disagree).

## Plan Before the First Run

1. **Write the full lyrics first**, `[End]` included. Improvising lyrics segment-by-segment mid-build produces theme drift you can't fix later.
2. **Mark segment boundaries at section seams** — end of a chorus, end of a bridge. Never mid-verse: the model finishes phrases badly when the handoff lands inside one.
3. **Budget the credits.** Each extend is one generation at full cost. Worked example: a 4-minute build ≈ 2 initial attempts + 3 extends = 5 runs, which exhausts a full free-tier day. Confirm the budget — or a paid plan — before starting.
4. **Validate the core first.** Do not begin a multi-extend build until an opening clip already lands (SKILL.md Traps).

## The Extend Loop

1. Generate the opening; evaluate both clips as two rolls of one prompt; pick the stronger.
2. Play it; note the timestamp where it degrades — melody drifts, mix collapses, vocal changes character.
3. Extend **from the last good moment**, not the clip's end — everything after the chosen point is discarded and regenerated.
4. In the extend's lyrics box, paste the next slice of the full lyrics with already-sung sections removed; leaving them in gets them sung twice.
5. Keep the style string **byte-identical** across segments. Change it only for a planned section shift (final chorus gains "orchestral, big finish") — one deliberate delta, never a rewrite.
6. Repeat until the `[End]` segment, then stitch with Get Whole Song.

## Crop and Replace

| Problem location | Tool |
|---|---|
| Bad part at a clip's start or end | Crop — also for isolating the good 30 seconds of a 2-minute roll before extending |
| Bad span in the middle of good material | Replace Section (in the editor, on versions that offer it) — regenerates the span, keeps both sides |
| Song needs to continue forward | Extend — the only tool that adds material |

- Crop before extend when the flaw sits inside otherwise good material: extending from before the flaw regenerates everything after it, including parts you liked.
- Song starts too abruptly: crop the current opening and regenerate the first segment with an `[Intro]` tag — there is no backward extend.

## Seams

Audible-join checklist, in order of likelihood:
1. Style string identical across the two segments?
2. Extension point at the end of a musical phrase, not mid-note?
3. Same clip lineage — every extend made from the chosen chain, not a sibling roll?

After Get Whole Song, listen across every join once at full attention. Remastering can smooth loudness jumps between segments; it cannot fix melodic mismatches — those need re-extending from before the bad join.

## Stitching Outside Suno

Get Whole Song covers the standard case. Reach for a DAW or `ffmpeg` when: joining clips from different generations (an instrumental intro made separately), cross-fading a loop, or trimming to an exact duration. Hand off with the `audio`/`ffmpeg` skills — that work is out of this skill's scope.
