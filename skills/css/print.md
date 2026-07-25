# Print and PDF Output

Browsers print through a different layout mode: paged, no viewport, no scrolling, and colors off by default. Most "the PDF looks broken" reports are one of the six items below.

## The Print Baseline

```css
@media print {
  @page { size: A4 portrait; margin: 15mm; }
  html { font-size: 11pt; background: none; color: #000; }
  nav, aside, .no-print, dialog, [popover] { display: none !important; }
  a[href^="http"]::after { content: " (" attr(href) ")"; font-size: .85em; word-break: break-all; }
  .card, tr, figure, blockquote { break-inside: avoid; }
  thead { display: table-header-group; }
}
```

- `@page` controls the sheet: `size` (`A4`, `Letter`, or explicit dimensions plus orientation) and `margin`. Browsers still let the user override both in the print dialog — never depend on your margin for content that must not be cut.
- Print units are physical: `pt`, `mm`, `cm` behave predictably; `vh`, `dvh`, and `vw` are meaningless on paper (the "viewport" is a page).
- Print styles inherit from screen styles unless overridden — including dark themes. Reset colors explicitly or your PDF is a black rectangle.

## Backgrounds and Color

- Browsers omit background colors and images by default. `print-color-adjust: exact` (with `-webkit-print-color-adjust` for older engines) forces them, and the user can still disable "background graphics".
- Never encode meaning in a background alone for print: zebra rows, colored status pills, and highlight bars vanish. Add borders, symbols, or text.
- Convert to a print-safe palette rather than printing your screen palette: light grey backgrounds below roughly 10% tint disappear on many printers; thin light-grey hairlines vanish entirely.

## Page Breaks

- Current properties: `break-inside: avoid`, `break-before: page`, `break-after: avoid`. The `page-break-*` names are legacy aliases and still work.
- `break-inside: avoid` on an element taller than one page is ignored — it cannot honor an impossible constraint; split the content instead.
- `orphans: 3; widows: 3` prevents single lines stranded at a page boundary; both apply only in paged media.
- Tables: `thead { display: table-header-group }` repeats headers on every page, `tfoot { display: table-footer-group }` repeats footers. Sticky positioning does nothing in print.
- Force a break between logical sections (`section { break-before: page }`) rather than inserting empty spacer elements, which reflow at any margin change.

## Layout Differences

- Multi-column screen layouts should collapse to one column: grid and flex work in print, but a 3-column grid at 180mm is unreadable. Reset containers to `display: block` for print.
- `position: fixed` elements print once per page in some engines and once at the top in others — set them to `static` or hide them.
- Overlays are invisible on paper but their scroll lock is not: an open modal often means a blank print. Print from a clean state, and hide `dialog`/`[popover]` in print CSS.
- Scroll containers clip: an `overflow: auto` region prints only its visible portion. Unset `overflow` and `max-height` in print for anything that must appear in full — this is the single most common cause of "half my table is missing".
- Images: `max-width: 100%` plus `break-inside: avoid`. Lazy-loaded images below the fold may not have loaded when printing starts; force `loading="eager"` for print-critical figures.

## Headers, Footers, Page Numbers

- Browsers add their own header/footer (URL, date, page number) and the user controls them; you cannot style or remove them from CSS.
- The Paged Media margin boxes — `@page { @bottom-right { content: counter(page) } }` — are supported by print engines like Prince and by Paged.js, NOT by browser print. Anything that needs designed running headers and real page numbers needs one of those, or server-side PDF generation.
- `counter(page)` outside a margin box does nothing in browsers either. Do not ship it and assume it works because it parsed.

## Testing

- Chromium print preview plus "Save as PDF" is the fastest loop; also emulate print rendering from DevTools (rendering panel) so you can inspect the print layout with the element inspector.
- Check both paper sizes if the audience is mixed: A4 and Letter differ enough to change break positions.
- Print at least three pages of real content — single-page tests never reveal break, repeated-header, or orphan bugs.
- Verify with background graphics both on and off, since that is a user setting.
