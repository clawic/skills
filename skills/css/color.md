# Color

Color spaces, mixing, gradients, and contrast mechanics. The WCAG ratios are canonical in SKILL.md — Accessibility Floor.

## Pick the Space on Purpose

| Space | Use it for | Why |
|---|---|---|
| `oklch()` | Palettes, tints/shades, anything generated | Lightness is perceptual: equal L reads as equally light across hues |
| `hsl()` | Legacy files, quick hand-tweaks | Lightness is not perceptual — `hsl(60 100% 50%)` yellow and `hsl(240 100% 50%)` blue both claim 50%, yet relative luminance is 0.93 vs 0.07 (~13×) and perceptual lightness L* 97 vs 32 (~3×) |
| `color(display-p3 …)` | Saturated brand accents on wide-gamut screens | Reaches colors sRGB cannot; clamps politely elsewhere |
| Hex / `rgb()` | Handoff values, third-party APIs | Universally understood, no math affordance |
| Default when unsure | `oklch()` | All engines since 2023, and the only one where a 10% lightness step is a 10% lightness step |

- `oklch(L C H)`: L is 0-1 (or a percentage), C is chroma with no fixed maximum (~0.37 is the sRGB edge, most brand colors sit at 0.1-0.2), H is degrees. Out-of-gamut combinations are gamut-mapped, not clipped per channel — you get a nearby color, not a muddy one.
- Ramps built by stepping L at constant C and H stay recognizably the same hue; the same exercise in HSL drifts (blues turn purple as they lighten).

## Deriving Colors Instead of Hardcoding

- `color-mix(in oklch, var(--brand) 80%, black)` — shade; swap `white` for a tint. The `in <space>` argument matters: mixing in sRGB darkens through grey, mixing in oklch keeps chroma.
- Relative color syntax: `oklch(from var(--brand) calc(l - 0.08) c h)` — one source token, derived hover/active/disabled states that survive a rebrand. All engines since 2024.
- Alpha variants without a second token: `color-mix(in oklch, var(--brand) 12%, transparent)` for tinted backgrounds.
- Do not derive semantic colors from each other in long chains; two derivations deep is the readability limit. Store the third one as its own token.
- `currentColor` is the cheapest theming primitive: SVG icons with `fill: currentColor` follow text color everywhere, including inside dark mode and forced colors.

## Opacity vs Alpha

- `opacity: .5` applies to the whole subtree, creates a stacking context, and makes text on top of the element unreadable in ways you cannot fix locally.
- `background: rgb(0 0 0 / 50%)` (or `oklch(… / 50%)`) affects one property, no stacking context, children stay opaque. Default to alpha in a color; reserve `opacity` for fading whole elements in and out.
- Contrast over a translucent overlay must be measured against the COMPOSITED result, not the token — the checker cannot see through your layers.

## Gradients

- Interpolation space changes the middle: `linear-gradient(in oklab, blue, yellow)` avoids the grey dead zone that sRGB interpolation produces between complementary hues; `in oklch longer hue` gives a rainbow sweep.
- Fading to nothing: write the transparent stop in the SAME color (`rgb(0 0 0 / 0)`), never the `transparent` keyword next to a colored stop. Modern engines premultiply and mostly hide this, but explicit stops are portable and readable.
- Banding on large, low-contrast gradients is a 8-bit quantization artifact: add intermediate stops, interpolate in oklab, or overlay a subtle noise texture. Increasing the gradient size makes it worse, not better.
- Color stop positions accept two values (`red 0 40%`) for hard stripes without duplicating stops.
- Animating a gradient means animating custom properties registered with `@property` — gradients themselves interpolate discretely.

## Contrast in Practice

- Ratio formula (WCAG 2.x): `(L1 + 0.05) / (L2 + 0.05)` where L is relative luminance; the +0.05 flare term is why dark-on-dark pairs score better than they look.
- Text over images always needs a scrim: a gradient overlay (`linear-gradient(rgb(0 0 0 / .6), transparent)`) or a text-shadow floor. "It looks fine on my photo" is a sample-size-of-one argument.
- Disabled controls are exempt from contrast minimums (1.4.3), but a disabled control nobody can read is still a support ticket — keep it above 3:1.
- Focus indicators need 3:1 against BOTH the component and the adjacent background — a white ring on white breaks on the second surface.
- Never signal state by hue alone (1.4.1): pair color with an icon, weight, underline, or text.

## System and Forced Colors

- `@media (forced-colors: active)`: the OS replaces your palette. Backgrounds become `Canvas`, text `CanvasText`, links `LinkText`, buttons `ButtonFace`/`ButtonText`. Test that borders, focus rings, and icon-only buttons still exist there — the common failure is a component whose only affordance was a background color.
- `forced-color-adjust: none` on the rare element that must keep its colors (a color swatch, a chart legend). Using it on whole components defeats the mode.
- `@media (prefers-contrast: more)` is the user asking for stronger separation: thicken borders and raise text contrast; it is not a request for pure black on pure white.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Building a ramp by nudging HSL lightness | Perceived lightness jumps between hues; the "same" step is not the same | Step `L` in oklch at constant C/H |
| `transparent` as a gradient stop | Interpolates through transparent black in older engines — grey fade | Same color with `/ 0` alpha |
| `opacity` to lighten a background | Fades the text inside it too, and creates a stacking context | Alpha in the background color |
| Checking contrast on the token, not the render | Overlays, blend modes, and translucency change the composited value | Measure the rendered pixels |
| `filter: invert(1)` as dark mode | Inverts photos and logos, wrecks brand color | Semantic tokens swapped per theme |
| Hardcoding hex in components | A rebrand becomes a find-and-replace across the codebase | Semantic token, derived with `color-mix` |
