# Documents — PDFs, Reports, Resumes, Print

Reading artifacts: multi-page, linear, often printed. Hierarchy spans pages, not views. Units go pt/mm when `platform` is print.

## Page Setup

- A4 (210x297mm) or US Letter (216x279mm) — match the audience's region; a Letter layout prints clipped on A4 trays and vice versa.
- Margins: 20-25mm (0.8-1in) all around for documents; resumes can tighten to 15mm. Bound documents add 5mm to the inner margin.
- Body: 10-12pt print (serif text face, `fonts.md`), 1.4-1.5 line-height, single column at 60-70ch — the same measure law as screens, different units.
- Running elements: page numbers outside bottom corner, document title or section in the header from page 2 on. A multi-page PDF without page numbers fails the first time someone cites it.

## Report Hierarchy

- Heading ladder: max 3 numbered levels (1 / 1.1 / 1.1.1); a fourth level means the structure needs flattening, not smaller headings.
- Headings bind downward: more space above than below (e.g. 24pt above, 8pt below) — same law as SKILL.md Spacing.
- Key findings surface twice: once in an executive summary (one page, findings as assertions), once in context. Nobody reads a 30-page report linearly.
- Tables and figures: numbered captions below figures, above tables; every one referenced from the text ("see Figure 3") or it's decoration.
- Pull quotes and callout boxes: one per 2-3 pages maximum — each interrupts the reading column.

## Resumes / One-Pagers

- One page per ~10 years of experience (heuristic, not law); recruiters scan top-third first — name, role target, and the strongest 2-3 facts live there.
- Single column beats creative two-column layouts for machine parsing (ATS) and human scanning alike; skills sidebars die unread.
- One typeface, 400/700, 10-11pt body, generous section gaps. Decoration budget: zero — alignment and spacing carry the professionalism.

## Print Production

- Raster images: 300 DPI at final printed size (a 100mm-wide photo needs ~1200px). Screen-resolution images look sharp on screen and muddy on paper.
- Bleed: artwork touching the page edge extends 3mm (0.125in) past it; keep text 5mm inside the trim line (safe zone).
- Color: convert to CMYK before handoff or confirm the printer accepts PDF/X with embedded profiles. Saturated RGB blues and greens shift dull in CMYK — soft-proof before approving.
- Blacks: body text 100K only (single black plate, no registration ghosting); large black fills use rich black (e.g. C60 M40 Y40 K100). Never rich black for text.
- Hairlines below 0.25pt drop out on many presses; minimum rule weight 0.5pt.
- Export: PDF/X-1a or PDF/X-4, fonts embedded, marks and bleed included when the printer asks for them.

## Digital-First PDFs

- Destined for screens: RGB, hyperlinked table of contents and cross-references, bookmarks panel populated for 10+ page documents.
- Landscape 16:9 pages for read-on-screen decks/reports beat portrait A4 scrolling in viewers.
- File size: compress images to ~150 DPI for screen-only PDFs; a 40MB attachment is a design failure.
- Tagged PDF (reading order, alt text) when the document is public-facing (`accessibility.md`).

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Designing print pages in RGB and converting last-minute | Color shifts surprise at proof stage; too late to redesign | Soft-proof CMYK from the start for print jobs |
| Body text in 9pt to fit a page count | Below print legibility comfort; readers skim harder | Cut content or add a page — same law as screens (never shrink to fit) |
| Screenshots at 72 DPI in a printed report | Muddy blocks where evidence should be | Recapture at 2x or vector-export charts |
| Centered headings over justified body | Two alignment systems fight (Core Rule 6 applies on paper) | Left-align both; justify only with hyphenation enabled |
| No safe zone on edge-bleed designs | Text trimmed off by cutting tolerance | 3mm bleed out, 5mm safe zone in |
