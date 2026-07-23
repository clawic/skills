# Covers, Personas, and Uploads — Reusing a Sound

The features that carry identity from one clip to another. Hosted APIs mostly do not expose them — this is browser/UI territory.

## Covers

- What it does: re-renders a clip (or uploaded audio) in a new style, keeping melody and lyrics.
- Use for: genre swaps ("the acoustic version"), turning a hummed voice memo into a produced track, salvaging a great melody trapped in the wrong production.
- Workflow: select the clip → Cover → new style string; leave lyrics untouched. A cover is another sampling roll (SKILL.md rule 2) — expect two variants, judge both.
- Calibration: the melody skeleton survives; vocal phrasing and arrangement shift. If a specific vocal nuance is the point, a cover will likely lose it — crop and extend around it instead.

## Personas

- What it does: captures the voice, delivery, and vibe of one clip and applies it to new songs — a consistent artist identity across tracks.
- Source clip choice decides everything: pick the cleanest, most representative clip — clear solo vocal, minimal reverb and effects. A persona made from a washed-out source reproduces the wash on every future song.
- Persona + identical style string + same Exclude Styles = consistent album sound; vary only lyrics between tracks.
- Availability and limits vary by plan and version — check the account before promising a persona-based project.

## Uploads

- Upload your own audio (voice memo, guitar riff, hummed melody), then extend or cover it.
- Rights rule: upload only audio you own outright. Uploading a commercial recording to cover it is a terms violation, not a workaround — moderation checks uploads too.
- Length and count caps vary by plan; verify at suno.com before designing a workflow around uploads.
- Melody-first composition: hum the melody → upload → Cover with a full style prompt. This reverses Suno's default text-first flow and is the reliable way to get a specific melody the text interface cannot describe.

## Consistent Multi-Song Projects

For albums, podcast series, or a client's recurring spots:

1. Lock one style string in the project file (`~/Clawic/data/suno/projects/`) and reuse it verbatim.
2. Lock the voice with a persona; keep Exclude Styles identical.
3. Vary per song via lyrics plus at most one deliberate style delta ("track 4: + strings").
4. Record every track's prompt, clip URL, and status in the project file — consistency you cannot reproduce is consistency you do not have.
