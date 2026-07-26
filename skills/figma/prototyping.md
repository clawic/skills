# Prototyping — Flows, Motion, and State

A prototype exists to answer a question. Three connected screens with real transitions answer more than thirty disconnected frames, and cost less to maintain.

## Scope First

- Name the question before wiring anything: "can a first-time user find checkout", "does this transition read as forward or back", "is the empty state understandable".
- Prototype the flow, not the screen inventory. Screens with no interaction belong in the design file, not the prototype.
- One flow per starting point, named after the question. Multiple named flows in one file beat one tangled graph.
- Delete wiring that no longer serves a question. Stale noodles are the reason nobody trusts the prototype.

## Interactions

| Trigger | Use for | Watch for |
|---|---|---|
| On click / tap | The spine of any flow | Overlapping hit areas from absolutely-positioned children |
| While hovering / while pressing | Desktop affordances, press states | Meaningless on touch — do not rely on it in a mobile test |
| On drag | Sheets, carousels, swipe-to-delete | Needs a matching layer in the destination to feel continuous |
| After delay | Loading states, toasts, auto-advance | The most common source of a prototype that "jumps on its own" |
| Key / gamepad | Keyboard flows, escape handling | Only fires in presentation view, not on the canvas |
| Mouse enter / leave | Tooltips, dropdown reveal | Prefer an interactive component so the behavior travels |

Actions worth knowing beyond Navigate: Open overlay, Swap overlay, Close overlay, Back, Scroll to, Open link, Set variable, Conditional.

## Smart Animate

Smart Animate matches layers **by name and by position in the hierarchy** across the two frames, tweens what matches, and crossfades what does not.

- A crossfade where you expected movement always means the match failed. Rename both layers identically and put them at the same depth in the tree.
- Matching is per-branch: `Card / Title` matches `Card / Title`, not `Content / Title`. Restructuring the destination breaks motion that used to work.
- Duplicate the source frame to build the destination. It guarantees names and hierarchy match, and takes less time than debugging a mismatch.
- Easing: ease-out for elements entering, ease-in for exiting, spring for anything the finger is directly manipulating. Durations in the 150-300 ms band read as responsive; above ~400 ms the prototype starts to feel slow in testing even when nobody says so.
- Interactive components animate with the same rules inside the variant set, which is why hover and press states are cheaper as variants than as frames.

## Variables in Prototypes

Prototype variables turn a click-through into a small state machine.

- **Set variable** on an interaction writes a value (a counter, a boolean, a selected tab). Bind text or visibility to it and the screen updates without a second frame.
- **Conditional** branches on a variable: `if cartCount > 0 → checkout, else → empty state`. One frame covers both paths.
- **Expressions** compute values (`cartCount + 1`), which is what makes counters, quantity steppers, and form validation demos possible without duplicating screens.
- The practical ceiling: anything needing real data, persistence across sessions, or network latency is a code prototype, not a Figma one.

## Overlays and Scroll

- Overlay position: Manual (place it yourself, on the source frame) or Centered / Top / Bottom. Manual is right for menus anchored to a trigger.
- `Close when clicking outside` is off by default and is the most commonly missing setting in modal demos.
- Swap overlay transitions between two overlays without closing — the multi-step modal pattern.
- Scrolling requires overflow set on the frame plus content taller than the frame. Nothing else makes it scroll.
- `Fix position when scrolling` pins headers, nav bars, and FABs. On an auto layout frame, the fixed child must be absolutely positioned.
- Scroll to a specific layer for anchor links and long-form navigation.

## Testing With It

- Test the sad paths or the test tests a lie: empty list, one item, hundreds of items, the longest string, no image, slow network, permission denied.
- Device frame and presentation settings change the perceived size — test at the real device scale, not zoomed to fit.
- Record sessions with a testing tool rather than watching over a shoulder; the moderator's presence changes where people click.
- Prototype links are share links: an "anyone with the link" prototype of unreleased work is a leak, and anyone who receives it can forward it. Restrict to the team, open the link for the specific review, close it after.

## Performance in Prototypes

- Heavy raster images stutter transitions long before they slow the editor. Downscale images used in animated frames.
- Long chains of Smart Animate across large frames compound: each transition re-diffs both trees.
- A prototype spanning many pages loads slowly on first open. Keep the tested flow's frames on one page.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Motion is a crossfade | Names or hierarchy do not match across frames | Duplicate the source to build the destination |
| Element jumps from the wrong place | Matched with the wrong same-named layer | Make names unique within the branch |
| Prototype advances on its own | Stray After-delay interaction | Audit the flow's interactions and remove it |
| Modal will not close | `Close when clicking outside` off, no close interaction | Enable it, and wire the close control |
| Nothing scrolls | Overflow not set, or content not taller than the frame | Set overflow scrolling; check content height |
| Header scrolls away | Not fixed, or fixed but in flow | Absolute position plus fix-position-when-scrolling |
| Hover states do nothing on mobile test | Hover triggers are desktop-only | Rebuild the state on press or as a variant |
| Transitions stutter | Full-resolution images in animated frames | Downscale the raster used in transitions |
| Counter demo needs 12 duplicate screens | Not using variables and expressions | One frame, a number variable, set-variable on tap |
