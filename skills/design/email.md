# Email — Design Inside HTML Email Constraints

Email clients are the most hostile rendering target in design: assume 2005-era CSS and design within it, not around it.

## Hard Constraints

- 600-640px body width — the safe rendering width across desktop clients; center it on a full-width background wrapper.
- Layout with tables (`role="presentation"`), not flexbox/grid: Outlook on Windows renders with Word's engine — no flex, no grid, no background images without VML fallbacks, unreliable padding on div/p.
- Inline the critical CSS on each element; `<style>` blocks are stripped or partially supported in several clients. Media queries work in major mobile clients but treat them as enhancement.
- Gmail clips messages over 102KB of HTML (shows "View entire message" and hides the rest — including the unsubscribe footer). Keep HTML weight well under; images don't count toward the limit.
- Web fonts render only in a minority of clients (Apple Mail, some others): declare them, but design so the fallback stack looks intentional (`fonts.md` metrics-compatible fallback).
- No JavaScript, no video autoplay, forms unreliable — every interaction is a link out.

## Layout That Survives

- Single column, 600px, stacking sections — multi-column layouts need Outlook conditional tables and still break; reserve two-column for simple image+text pairs that can stack on mobile.
- Font floor 14px in email (16px preferred); line-height set in px (e.g. 16px/24px) — unitless line-height is inconsistent across clients.
- Buttons: padded table-cell with background color and the link filling it ("bulletproof button"), minimum 44px tall touch target. An image-as-button vanishes when images are blocked.
- Section spacing via padded cells or spacer rows on the 8px scale; margin is the least reliable property in email CSS.

## Images

- Many clients block images by default: the email must communicate with images off. Every image gets styled alt text; the message and CTA are HTML text, never baked into an image.
- 2x resolution, width/height attributes set in HTML (prevents layout jump), file weight compressed — image-heavy emails trip spam heuristics and mobile data patience.
- Full-image emails ("just send the flyer as a JPEG") fail on: blocked images, spam scoring, accessibility, and text search. Always rebuild as HTML.

## Dark Mode in Email

- Clients disagree: some invert colors (Outlook variants), some respect your `prefers-color-scheme` styles (Apple Mail), some do nothing. Design colors that survive inversion: avoid pure #FFF backgrounds (inverts to harsh #000), avoid text baked into images (won't invert — halos on inverted backgrounds).
- Logos: transparent PNG with a subtle outline or padding-safe version so forced-dark backgrounds don't swallow a dark logo.

## Hierarchy for the Inbox

- The design starts before the open: sender name, subject (~40 chars survive mobile truncation), and preheader text (the first hidden/visible text, ~80 chars) are the artifact's rank-1 — set the preheader deliberately or the first `<td>`'s content leaks into it.
- One message, one CTA per email — the landing-page hero rule (`landing-pages.md`) compressed further. The CTA appears within the first screen and repeats near the end.
- Footer is legally load-bearing: sender address and a working unsubscribe link — designed small (rank-3, muted) but present and tappable.

## Testing

Render-test before sending: Outlook Windows, Gmail (web + app), Apple Mail (light + dark), one Android client. Send to a seed address and read it on a phone with images blocked — that's the worst honest case. No preview tool replaces one real send.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Reusing a web page's HTML/CSS | Flex/grid/external CSS collapse in Outlook and Gmail | Rebuild as inline-styled tables at 600px |
| Message text inside the hero image | Blocked images = blank email; no dark-mode adaptation | HTML text over a background color; images as support |
| One giant JPEG as the email | Spam scoring, blocked-image blank, zero accessibility | HTML structure with compressed supporting images |
| Ignoring the preheader | Inbox shows "View in browser \| Unsubscribe" as the preview | Craft ~80 chars of preheader continuing the subject |
| Testing only in the browser preview | Browser is the ONE client that renders everything | Seed-send to real clients incl. Outlook and images-off |
