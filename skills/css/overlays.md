# Overlays — Modals, Popovers, Tooltips, Dropdowns

Anything that floats above the page. The default is the top layer, not a bigger z-index (SKILL.md Core Rule 7); the stacking-context walk-up for everything that stays in the page is in SKILL.md — Stacking Contexts.

## Pick the Mechanism First

| Need | Mechanism | Why |
|---|---|---|
| Blocking dialog (confirm, form) | `<dialog>` + `showModal()` | Top layer, native focus trap, Esc to close, `::backdrop`, background made inert |
| Non-blocking overlay (menu, tooltip, toast) | `popover` attribute | Top layer, light-dismiss, no focus trap, no inert |
| Content that must stay in flow | Positioned element + `isolation: isolate` on the section | No top layer needed; keep the z-index war inside one component |
| Toast stack | Popover elements in a fixed-position container | Top layer order follows open order; no z-index bookkeeping |
| Anything else floating | Start with popover, escalate to modal dialog only when interaction must be blocked | Cheapest mechanism that satisfies the interaction |

Top-layer elements ignore ancestor stacking contexts, ancestor `overflow: hidden`, and ancestor `transform` — the three reasons overlays get clipped or buried. A modal that still renders behind the header is a modal that was never promoted.

## Modal Dialog

```css
dialog { border: 0; padding: 0; max-width: min(90vw, 32rem); }
dialog::backdrop { background: rgb(0 0 0 / .5); }
dialog:not([open]) { display: none; }
```

- `showModal()` gives the focus trap and inert background; `show()` and manual `[open]` do not — a "modal" opened by toggling the attribute is a non-modal box with a fake overlay.
- Background scroll is not locked for you: add `overflow: hidden` on `<html>` while open, and `scrollbar-gutter: stable` on `<html>` at all times so removing the scrollbar does not shift the page.
- `::backdrop` no longer inherits from the page root in current engines but does inherit from the originating element — set variables on the dialog itself if the backdrop needs tokens.
- Animate in and out with `@starting-style` plus `transition: … overlay .2s allow-discrete` — without `overlay` in the list the element leaves the top layer instantly and the exit animation is never seen.
- Full-bleed sheets: `dialog { margin: 0; max-height: 100dvh; height: 100dvh; margin-inline-start: auto }` for a right-side drawer; the UA centers dialogs by default via auto margins.

## Popover

- `popovertarget` on the trigger and `popover` on the panel gives open/close, light dismiss, and top-layer promotion with zero JS.
- `popover="auto"` closes on outside click and Esc and closes other auto popovers; `popover="manual"` closes only when you say so (toasts).
- Nested popovers stay open together only when the child is inside the parent's DOM subtree or points at it with `popovertarget` — otherwise opening the child closes the parent.
- `:popover-open` is the styling hook. `::backdrop` also works on popovers for a scrim without blocking.
- Popover does not manage focus: move focus into the panel when it opens if it contains controls, and return it to the trigger on close.

## Positioning Against a Trigger

- Anchor positioning is the CSS-native answer:

```css
.trigger { anchor-name: --menu-btn; }
.menu {
  position: absolute;
  position-anchor: --menu-btn;
  position-area: block-end span-inline-start;
  position-try-fallbacks: flip-block, flip-inline;
  max-height: calc(anchor-size(height) * 3);
}
```

- `position-try-fallbacks` is what keeps a menu on screen near the viewport edge — without it, anchored elements clip exactly like hand-positioned ones.
- Engine support is uneven (Chromium first, Safari following, Firefox trailing): gate with `@supports (position-area: block-end)` and ship a plain absolute-positioned fallback, or use a positioning library when the design cannot tolerate the difference.
- Without anchor positioning, the portable pattern is a `position: relative` wrapper plus an absolutely positioned panel — which reintroduces clipping from any `overflow: hidden` ancestor. That clipping is the reason to prefer top layer plus anchoring.

## Tooltips

- `title` is the only zero-cost tooltip and it is unstyleable, delayed, and invisible on touch. Anything designed is a component.
- Hover-only tooltips fail keyboard and touch users: show on `:hover` AND `:focus-visible`, and make the content available another way on touch.
- WCAG 1.4.13 requires hoverable, dismissible, persistent tooltips: the pointer must be able to move onto the tooltip without it vanishing, Esc must close it, and it must not disappear on its own.
- Never put essential or interactive content in a tooltip — link and button targets inside a hover panel are unreachable for many users.

## Dropdown and Menu Clipping

1. Panel cut off at a parent edge → an ancestor has `overflow: hidden`/`auto`. Promote to the top layer, or (if it must stay inline) remove the clipping ancestor's overflow.
2. Panel behind a later sibling → stacking context (walk-up in SKILL.md).
3. Panel jumps when it opens → the trigger row has no reserved space; position absolutely so the panel never participates in layout.
4. Panel inside a `transform`ed ancestor with `position: fixed` → the transform is the containing block; fixed became absolute.
5. Menu inside a scroll container → it scrolls away with the content unless it is in the top layer or repositioned on scroll.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `z-index: 999999` on the overlay | Loses to any sibling context and clutters the scale forever | Top layer |
| `overflow: hidden` on `body` for scroll lock | iOS loses the scroll position and the page jumps on close | Lock `<html>`, keep `scrollbar-gutter: stable`, restore scroll on close |
| Custom div "modal" | No focus trap, no Esc, background still tabbable | `<dialog>.showModal()` or `inert` on the background |
| Backdrop as a sibling div | Sits in the page stacking order and clips like everything else | `::backdrop` |
| Tooltip on `:hover` only | Unreachable by keyboard, sticky on touch | `:hover, :focus-visible` + `@media (hover: hover)` |
| Animating the overlay without `allow-discrete` | Exit animation never plays; element vanishes | `transition-behavior: allow-discrete` + `overlay` |
| Dialog sized with `vh` | Mobile toolbars overlap the buttons at the bottom | `dvh`/`svh` |
