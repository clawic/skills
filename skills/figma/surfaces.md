# FigJam, Slides, and the Adjacent Surfaces

Figma is several editors sharing one account. Each has its own object model, its own plugin surface, and its own seat economics. Using the wrong one is the most common cause of a file nobody can maintain.

## Choosing the Surface

| Deliverable | Surface | Why |
|---|---|---|
| Workshop, brainstorm, retro, journey map | FigJam | Infinite messy canvas, voting and timer built in, cheapest seat |
| Flow diagram, service blueprint, IA map | FigJam | Connectors auto-route and survive moving the boxes |
| Product screens, components, tokens | Design file | The only surface with auto layout, variants, and variables |
| Presentation, review deck, readout | Slides | Deck structure, presenter view, and it consumes the design system |
| Marketing page or microsite output | The site-building surface | Publishes; not a source of truth for the product system |
| AI-generated first draft of an interface | The generative surface | Exploration input, never the artifact you hand off |
| Illustration and freehand vector work | The drawing surface | Deeper vector tooling than a design file needs |
| Social and marketing asset variants at volume | The templating surface | Bulk variants from one template |

Default rule for everything after the third row: those surfaces produce **inputs and outputs**, never the source of truth. The design system lives in a design file, and anything generated elsewhere gets rebuilt there before it ships.

## FigJam Mechanics Worth Knowing

- Sticky notes carry an author; turn author names on for accountability and off for anonymous divergent phases.
- Voting runs a timed dot-vote with a set number of votes per person — the fastest way to end a converging discussion that has stalled.
- The timer is the facilitation tool that actually changes outcomes: a visible countdown stops the loudest person from consuming the round.
- Connectors attach to objects, not coordinates, so a rearranged diagram stays connected. This is the reason to diagram in FigJam rather than drawing arrows in a design file.
- Sections group and can be collapsed, which is how a three-hour workshop board stays navigable afterwards.
- Templates matter more here than anywhere else: a blank infinite canvas produces a blank workshop.
- Widgets embed live objects — ticket trackers, checklists, polls — and persist state for everyone in the file.

## Moving Between FigJam and Design Files

- Copying from FigJam into a design file drops FigJam-only objects (stickies, connectors, stamps, widgets). What survives is plain shapes, text, and images.
- The correct handoff is a link, not a copy: keep the workshop in FigJam, keep the screens in a design file, and cross-link both from the cover pages.
- Synthesize before you migrate. A workshop board pasted into a design file is clutter; the three decisions it produced, written on the cover page, are the artifact.

## Slides

- A deck built from the design system's components stays on-brand and updates when the library does; a deck built from freehand rectangles diverges immediately.
- Presenter view carries notes and a timer; interactive elements let a live demo run inside the deck instead of switching apps mid-review.
- Template discipline is the whole game: define title, section, content, and full-bleed layouts as components before writing slide one.
- Decks are not archives. Put the decision in a document or on the file cover; the deck is the delivery vehicle for one meeting.

## The Newer Surfaces

Figma has added generative and publishing surfaces (site building, AI-generated interfaces, templated marketing assets, and a dedicated drawing tool). They are recent, they change quickly, and their capabilities and plan gating are the least stable facts in this skill — verify the current state before building a workflow on one.

The judgment that holds regardless of what shipped this quarter:

- **Generated output is a draft.** It is fast at exploring layout ideas and slow at everything that makes an interface correct: real content lengths, states, accessibility, and token discipline.
- **Publishing surfaces bypass engineering review.** Decide deliberately who is allowed to publish and what the rollback is, before the first page goes live.
- **Nothing generated becomes canonical without a rebuild.** The design system is the source of truth; anything that did not come out of it gets rebuilt into it or stays a mock.
- **Plan gating is where the surprise lands.** Check the seat and tier requirements before committing a team workflow to a new surface.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Diagram arrows detach when boxes move | Drawn in a design file instead of FigJam | Rebuild with connectors in FigJam |
| Workshop board pasted into the design file as clutter | Copied instead of synthesized | Link the board; write the decisions on the cover |
| Stickies vanished on paste | FigJam-only objects do not cross over | Export an image, or keep the board where it is |
| Deck goes off-brand within a week | Freehand slides instead of components | Build title, section, content layouts as components |
| Nobody can find the decision after the workshop | It only exists on the board | Cover-page summary with owner and date |
| Generated page shipped without review | Publishing surface with no gate | Define who publishes and how to roll back |
| Team blocked on a surface they cannot access | Seat or tier gate | Check plan requirements before adopting the surface |
| Workshop ran long and converged on nothing | No timer, no voting | Timed rounds plus a dot vote |
