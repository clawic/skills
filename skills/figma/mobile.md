# Mobile and Platform Specs

Web habits break on native. The differences that matter are units, safe areas, system UI, and the components you should not draw at all.

## Units and Density

| Platform | Design unit | Export scales |
|---|---|---|
| Web | CSS px | 1x, 2x for raster; SVG for vector |
| iOS | pt | @1x, @2x, @3x |
| Android | dp | mdpi 1x, hdpi 1.5x, xhdpi 2x, xxhdpi 3x, xxxhdpi 4x |

- Design at 1x logical size. Designing at 2x and dividing produces half-pixel values across the whole file and a set of assets that never quite align.
- 1 pt and 1 dp are the same idea: a density-independent unit that the OS multiplies. Values are directly comparable across iOS and Android, which is why one spacing scale can serve both.
- Android's 1.5x bucket punishes odd numbers: a 25 dp asset renders at 37.5 px on hdpi. Keep base dimensions even and preferably divisible by 4.
- Use Figma's device presets for frame sizes rather than remembered numbers — device dimensions change every hardware generation, and the presets track them.

## Safe Areas and System UI

Reserve these regions before laying anything out; retrofitting them shifts every screen.

- **Top**: status bar, and on notched devices the cutout region. Content can run under it; controls cannot.
- **Bottom**: the home indicator and gesture area on gesture-navigation devices. A bottom bar must sit above it, and a swipe-to-dismiss control placed there fights the system gesture.
- **Sides**: on landscape and on curved-edge devices, inset content from the physical edge.
- **Keyboard**: it covers roughly the bottom third and the exact height varies by device, language, and whether a suggestion strip is showing. Spec what scrolls, what stays pinned, and which field must remain visible.
- **Android navigation bar**: three-button versus gesture navigation changes the bottom inset. Design for the taller of the two.

Show safe areas as a locked overlay layer on the template frame so nobody has to remember them.

## Native Components You Should Not Redraw

Date pickers, time pickers, system share sheets, permission dialogs, keyboards, context menus, and the platform's own alerts are OS-provided. Redrawing them produces a mock that cannot be built as designed and hides real behavior (dynamic sizing, localization, accessibility already handled).

Represent them as a labelled placeholder marked "native" and spec the parameters instead: which picker mode, which options, what the confirm action does.

## Type and Scaling

- Both platforms scale text at the user's request (Dynamic Type on iOS, font scale on Android). A meaningful share of users run larger-than-default text.
- Design the largest supported text state for at least the primary screens. Fixed-height rows, single-line tab bars, and side-by-side buttons break first.
- Truncation is a decision, not a fallback: state which strings truncate and which wrap, per component.

## Touch Targets and Gestures

- 44 × 44 pt on iOS, 48 × 48 dp on Android, and never less than 24 × 24 CSS px anywhere.
- Reserve system gesture zones: back-swipe from the screen edge on both platforms, and the bottom home gesture. A horizontal carousel that starts at the very edge conflicts with edge-back.
- Long-press, swipe-to-delete, and pull-to-refresh have platform-standard behavior; deviating from it costs discoverability for no gain.

## Adaptive Layouts

- Phone → large phone → tablet → foldable is a size continuum, not four fixed designs. Define what rearranges (single column becomes two, a list-detail becomes side-by-side) and what merely resizes.
- Foldables change size mid-session and can be half-open. State whether the app reflows or restarts.
- Tablets are not stretched phones: a full-width form on a tablet is a usability defect. Clamp measure the same way you would on desktop.

## Assets and Icons

- Ship vector where the platform accepts it; density buckets exist only for raster.
- App icons and adaptive icons follow platform-specific templates with their own safe zones — take the current template rather than approximating.
- Set every density as a separate export row on the icon or image component; one selection then produces the whole set, correctly suffixed.

## What to Spec Beyond the Screens

- Scroll behavior per region, and which elements pin.
- Safe-area insets applied per screen.
- Status bar style (light or dark content) per screen.
- Keyboard type per field (email, numeric, phone) and the return-key action.
- Loading and offline behavior — mobile networks fail in ways desktop mocks never show.
- Haptics, where they fire and how strongly.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Bottom bar sits under the home indicator | Safe area not reserved | Add the inset to the template; lock it as an overlay |
| Assets land on fractional pixels on Android | Odd base dimensions against the 1.5x bucket | Even values, divisible by 4 |
| Layout collapses at large text sizes | Only the default size was designed | Design the largest-text state for primary screens |
| Carousel fights the back gesture | Horizontal drag starting at the screen edge | Inset the drag region from the edge |
| Date picker cannot be built as drawn | A native component was redrawn | Placeholder marked native, plus parameters |
| Form unusable on tablet | Full-width layout on a wide screen | Clamp the measure; consider list-detail |
| Field hidden behind the keyboard | Keyboard avoidance never specced | State what scrolls and what stays visible |
| Everything is a half pixel off | File designed at 2x | Rebuild at 1x logical size |
