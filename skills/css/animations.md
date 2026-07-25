# Transitions and Animations

Mechanics: what interpolates, what doesn't, and how to make enter/exit states work. Frame budget and compositing cost are Core Rule 2 in SKILL.md.

## What Actually Interpolates

- A transition needs two resolved values. `auto`, `none`, `fit-content`, and intrinsic keywords have no numeric form: `height: 0 → auto` does nothing. Portable path is the `grid-template-rows: 0fr → 1fr` wrapper on a parent whose child carries `min-height: 0; overflow: hidden`; `interpolate-size: allow-keywords` is Chromium-only.
- Discrete properties (`display`, `visibility`, `content-visibility`, unregistered custom properties) snap at 50% unless you opt in with `transition-behavior: allow-discrete`.
- Custom properties interpolate only after `@property` registration with a `syntax` — this is what makes animated gradients and shadow colors possible.
- Shorthands transition per longhand: `background: red → url(...)` interpolates the color and swaps the image discretely.
- Element must be in the layout tree when the transition starts. Newly inserted elements need `@starting-style`.

## Enter and Exit Without JS

```css
.toast {
  opacity: 1; translate: 0 0;
  transition: opacity .2s, translate .2s, display .2s allow-discrete;
}
@starting-style { .toast { opacity: 0; translate: 0 8px; } }  /* enter-from */
.toast[hidden] { opacity: 0; translate: 0 8px; display: none; } /* exit-to */
```

- `@starting-style` supplies the before-open value; without it the element appears already-open. All engines since mid-2024.
- Popovers and modal dialogs also need `overlay` in the transition list — `transition: opacity .2s, overlay .2s allow-discrete` — otherwise the element leaves the top layer instantly and the exit animation is invisible.
- Exit animations are the half everyone forgets: verify by closing, not only by opening.

## Timing

- Durations that read as intentional in UI work: 100-200ms for state feedback (hover, press, checkbox), 200-300ms for entering elements, 150-200ms for exits (leaving should be faster than arriving), 300-500ms for large or full-screen transitions. Material Design's motion guidance is the common ancestor of these ranges; treat them as defaults to adjust by distance, not laws.
- Scale duration with distance, not with importance: an 8px nudge at 300ms looks broken; a full-width panel at 100ms looks like a cut.
- Easing defaults: `ease-out` for entering (fast start, settle), `ease-in` for exiting, `ease-in-out` for moves that start and end on screen. The CSS `ease` default is asymmetric and fine for hovers, wrong for entrances.
- `linear()` approximates springs and bounces with sampled points — it is a real easing function, not a hack, and keeps the animation on the compositor.
- Stagger with one variable, not N rules: `.item { transition-delay: calc(var(--i) * 40ms) }` and set `--i` in markup. Keep total stagger under ~300ms or the last item feels broken.

## Keyframes

- `animation-fill-mode: forwards` holds the last frame; `both` also applies the first frame during the delay. Default `none` snaps back and looks like a bug.
- `animation-composition: add` layers an animation on top of an existing transform instead of replacing it — the clean way to combine a hover scale with a running float.
- Pause instead of destroying state: `animation-play-state: paused`.
- `steps(n, jump-none)` for sprite sheets and typewriter effects.
- Naming a shorthand you later override: `animation: spin 1s linear infinite` resets every unlisted sub-property (delay, fill-mode) to initial. Order the shorthand first, longhand overrides after.
- Infinite background animation is the one legitimate case for permanent `will-change` — a spinner that never stops.

## Reduced Motion, Correctly

```css
@media (prefers-reduced-motion: no-preference) {
  .card { transition: transform .2s; }
}
```

- Opt in rather than override: styles written inside the no-preference query cannot leak.
- Never `* { animation: none !important }`. Killing animations outright means `animationend`/`transitionend` never fire, and components that clean up in those handlers hang forever. If you must blanket-disable, set `animation-duration: .01ms !important; transition-duration: .01ms !important` so events still fire.
- Reduced motion means less vestibular motion, not zero feedback: keep opacity and color changes, drop parallax, large translations, spins, and autoplaying loops.

## View Transitions

- Same-document: wrap the DOM change in `document.startViewTransition(cb)`; CSS controls the animation via `::view-transition-old(name)` / `::view-transition-new(name)`.
- `view-transition-name` must be unique among elements rendered at the same time — two cards sharing a name abort the transition entirely. Assign it to the one element that morphs, remove it after.
- `view-transition-class` styles families of named elements without repeating rules.
- Cross-document transitions need `@view-transition { navigation: auto }` in both documents and same-origin navigation.
- Engine support is uneven (Chromium first, Safari following, Firefox trailing) — the API degrades to an instant swap, so it is safe as progressive enhancement and unsafe as the only way to see a state change.

## Scroll-Driven Animations

- `animation-timeline: scroll()` ties progress to a scroll container; `view()` ties it to an element's visibility within the scrollport. Both replace scroll-listener JS and run off the main thread.
- Only `animation` works, never `transition` — there is no time to transition against.
- `animation-range: entry 0% cover 40%` bounds the active window; without it, the animation spans the whole range and looks static.
- Chromium shipped this first and other engines trail — build it so the no-timeline state is the final state (`animation-fill-mode: both` plus sane defaults), then it degrades to "already visible".

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `transition: all` | Picks up properties added later, including layout ones; theme swaps become visible sweeps | Enumerate properties |
| Animating a `filter: blur()` over a large area | Repaints the whole region every frame | Animate opacity of a pre-blurred layer |
| Transition on the parent's `transform` plus the child's | Transforms compose; the child moves twice as far as designed | Animate one level, or use individual transform properties |
| Hover animation on touch devices | Hover sticks after tap and the animation replays on scroll | Wrap in `@media (hover: hover)` |
| Long `animation-delay` for orchestration | State and timing drift the moment anything reorders | Stagger with a calc'd delay bounded to ~300ms |
| Animating `box-shadow` for elevation | Shadow repaint is expensive at large blur radii | Animate the opacity of a pseudo-element carrying the shadow |
