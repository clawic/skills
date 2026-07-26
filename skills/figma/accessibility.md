# Accessibility — What the File Must Carry

The visual layer cannot express focus order, semantics, or state changes. If the file does not carry them, the engineer invents them, and the audit finds the invention months later. This file covers speccing accessibility inside Figma; running the audit against the built product is a separate job.

## Contrast

| Content | Minimum ratio |
|---|---|
| Body text | 4.5:1 |
| Large text (≥24 px regular, or ≥18.66 px bold) | 3:1 |
| UI component boundaries, icons carrying meaning, focus indicators | 3:1 |

- Check the bound values in **every mode that ships**, not only Light. A palette that passes in Light routinely fails in Dark, because a semantic token points at a different primitive there.
- Check the token, not the screenshot. Contrast is a property of the pair of variables; verifying it once per pair covers every screen that uses them.
- Disabled controls are exempt from the contrast minimum in the specification, but a disabled control nobody can read is still a usability defect — keep it legible.
- Text over images and gradients needs a scrim with a stated opacity, not a hope. Spec the scrim as part of the component.

## Beyond Color

- Never encode meaning in color alone. Error red needs an icon or text; a required field needs a marker; a chart series needs a shape, a pattern, or a direct label.
- Test the palette against the common color-vision deficiencies with a simulation tool. Red/green pairs and blue/purple pairs are the usual failures.
- Focus indicators are content, not decoration: spec the visible focus treatment (offset, thickness, color) as a component state, or engineers will use the browser default and someone will remove it in CSS.

## Touch and Pointer Targets

| Context | Minimum |
|---|---|
| WCAG 2.2 minimum target size | 24 × 24 CSS px |
| iOS guidance | 44 × 44 pt |
| Android guidance | 48 × 48 dp |

- The target is the hit area, not the icon. A 16 px icon inside a 44 pt padded frame is correct; a 44 pt icon is not.
- Adjacent targets need spacing between hit areas, or the tap lands on the neighbour. Build the padding into the component so it cannot be forgotten.

## What to Annotate

Annotations attach to nodes and travel with the file. These are the items code cannot infer:

| Item | What to write |
|---|---|
| Focus order | A numbered sequence per screen, including where focus goes after a modal closes |
| Role and name | The accessible role and the accessible name for anything not plain text |
| Heading structure | Which text nodes are headings, and their level |
| State announcements | What a screen reader should say when a value changes, a filter applies, or content loads |
| Error association | Which message belongs to which field, and when it fires |
| Landmarks | Nav, main, complementary regions on a page layout |
| Alt text | The actual string for every meaningful image; explicitly mark decorative ones |
| Motion | What animates, and the reduced-motion fallback |
| Keyboard | Shortcuts, escape behavior, arrow-key navigation inside composite widgets |

## Reflow and Resize

- Text must remain usable at 200% zoom, and content must reflow to a 320 px equivalent width without a second scroll axis. Both are checkable in Figma with the drag sweep plus a scaled type test.
- Design one screen at the largest supported text size, not just the default. Fixed-height rows and single-line tabs are what break first.
- Line length: an over-wide measure hurts everyone and hurts dyslexic readers most. Clamp the text container, not the text node.

## Working Method

1. Build the accessible version first — contrast-passing tokens, real focus states, padded targets — rather than retrofitting.
2. Annotate as you build. Annotation written at handoff time is written from memory and misses the interactions.
3. Run a simulation and contrast plugin pass per screen before Ready for dev.
4. Keep a standing a11y checklist frame in the file so reviewers check the same items every time.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Contrast passes in Light, fails in Dark | Semantic token points at a different primitive per mode | Re-check every pair per mode; adjust the Dark primitive |
| Audit reports missing focus indicators | Focus state never designed | Add a focus variant to every interactive component |
| Screen reader announces nothing on filter change | State announcement never specced | Annotate the live-region behavior |
| Errors read as unattached text | No field association specced | Annotate which message belongs to which input |
| Taps land on the wrong control | Hit areas smaller than the minimum, or adjacent | Pad inside the component; separate hit areas |
| Chart unreadable for color-blind users | Meaning encoded in hue alone | Add shape, pattern, or direct labels |
| Layout breaks at 200% zoom | Only tested at default type size | Design the largest-text state |
| Images have no alt text in the build | Never provided in the file | Write the string per image; mark decorative ones |
