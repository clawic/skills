# Scrolling and Scroll Containers

Scroll containers, snap, scrollbars, and the bugs that come from creating a scrollport by accident.

## Who Is Actually Scrolling

Every `overflow: auto|scroll|hidden` box becomes a scroll container, and that decides the behavior of everything inside it:

- `position: sticky` sticks to the nearest scrollport, not the page — a sticky header inside an `overflow: hidden` wrapper never moves.
- Anchor links (`#section`) scroll the nearest scrollport that can move; if the wrong element scrolls, an ancestor took the job.
- Viewport units always mean the viewport, never the scroll container — size children of a scroller with `%` or `container` units instead.
- `overflow: clip` clips WITHOUT creating a scroll container: the correct tool when you only wanted to cut off overflow, and the only value that leaves `position: sticky` anchored to the page scroller. `overflow-x: clip` with `overflow-y: visible` is the one legal mixed pair, so it works on `html` (Safari 16+).
- The document scroller is `<html>`, not `<body>` — set `scroll-behavior`, `scroll-padding`, and `scrollbar-gutter` on `html`.

## Anchor Links Land Under the Sticky Header

- Global fix, once: `html { scroll-padding-top: var(--header-h) }` — every anchor jump and `scrollIntoView()` respects it, including browser back/forward restoration.
- Per-element fix: `scroll-margin-top` on the targets themselves; use it for the exceptions, not as the general rule.
- Focus follows the scroll: after a skip link, the target needs `tabindex="-1"` or focus stays behind — a CSS-adjacent bug that looks like a scroll bug.
- Smooth scrolling is a motion preference: `@media (prefers-reduced-motion: no-preference) { html { scroll-behavior: smooth } }`.

## Scroll Snap

```css
.carousel {
  display: flex; gap: 1rem;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  scroll-padding-inline: 1rem;
  overscroll-behavior-x: contain;
}
.carousel > * { scroll-snap-align: start; flex: 0 0 min(80%, 20rem); }
```

- `mandatory` always lands on a snap point; `proximity` snaps only when close. Use `proximity` whenever items can be taller or wider than the scrollport — with `mandatory`, content between snap points becomes unreachable, and users cannot stop mid-item.
- `scroll-snap-stop: always` prevents fast flicks from skipping several cards — required for anything paginated.
- `scroll-padding` on the container offsets snap positions past sticky chrome; `scroll-margin` does it per item.
- Snap containers still need keyboard access: the container itself must be focusable (`tabindex="0"`) or contain focusable children, plus visible previous/next controls for pointer users who cannot swipe.
- Snap plus `scroll-behavior: smooth` plus a JS scroll handler fights itself; pick the declarative path.

## Overscroll and Chaining

- `overscroll-behavior: contain` on modals, drawers, chat panes, and any inner scroller: reaching the end stops instead of scrolling the page behind it.
- `overscroll-behavior: none` additionally kills pull-to-refresh and the rubber-band effect — justified on app-like full-screen surfaces, hostile on documents.
- `overscroll-behavior-y: contain` alone is the usual answer for vertical panels; keep horizontal chaining so trackpad gestures still work.

## Scrollbars

- Standard properties: `scrollbar-width: thin | none` and `scrollbar-color: <thumb> <track>`. Broad engine support arrived across 2024-2025 — for older Safari, `::-webkit-scrollbar` remains the fallback, written as a separate rule (never comma-joined with the standard ones).
- `scrollbar-gutter: stable` on `html` (and on any container that toggles content) removes the classic centered-layout jump when a scrollbar appears. `both-edges` keeps symmetric padding.
- Hiding scrollbars entirely (`scrollbar-width: none`) removes an affordance: only acceptable when another visible cue exists (arrows, dots, edge fades) — otherwise nobody knows the region scrolls.
- Overlay scrollbars (macOS default, most touch devices) mean your "scrollbar space" assumptions are wrong on half the devices: never compute layout from an assumed scrollbar width; `100vw` includes the classic scrollbar and is the standard cause of a 15px horizontal scroll on desktop.

## Scroll Performance

- `background-attachment: fixed` repaints the painted area on every scroll frame and is the most expensive common scroll effect — use a fixed-position layer behind the content instead.
- `content-visibility: auto` with `contain-intrinsic-size` on long lists of sections skips offscreen rendering.
- Scroll listeners that read geometry force synchronous layout every frame; prefer `IntersectionObserver`, scroll-driven animations (`animation-timeline: scroll()`), or `position: sticky`.
- `overflow-anchor` is on by default and preserves position when content loads above the viewport. Set `overflow-anchor: none` only on containers where your own virtualization does that job — disabling it globally reintroduces the jumping it was invented to fix.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `overflow-x: hidden` on `body` to stop sideways scroll | Makes the root a scroll container, breaks sticky and anchor behavior, hides the real culprit | Find the overflowing element first; if it genuinely cannot be removed, `html { overflow-x: clip }` — no scroll container, sticky survives |
| `scroll-snap-type: mandatory` on tall content | Content between snap points cannot be read | `proximity` |
| Custom scrollbars everywhere | Overrides OS accessibility settings and shrinks the drag target | `scrollbar-width: thin` at most, keep the target usable |
| `scroll-behavior: smooth` unconditionally | Vestibular trigger; also slows down power users | Wrap in `prefers-reduced-motion: no-preference` |
| `100vw` on a full-bleed section | Includes the classic scrollbar width; adds horizontal scroll | Give the page grid a full-bleed track and let the section span it |
| Nested scrollers with no `overscroll-behavior` | Inner scroller hands off to the page mid-gesture | `contain` on the inner one |
