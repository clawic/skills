# On-Page — Titles, Snippets, Headings, URLs, Images

On-page work is the cheapest lever in SEO: no engineering, no links, effect visible in weeks. It is also where most of the folklore lives, so each rule below states what it actually buys.

## Title Tag

- 50-60 characters. Google truncates by pixel width (~600px on desktop), so wide capitals truncate earlier than the character count suggests. Overflow is not a penalty; it just wastes the tail.
- Lead with the term the searcher typed. The first words survive truncation on mobile and carry the match.
- One brand mention, at the end, separated with `|` or `-`. Brand first only when brand is the query.
- Do not repeat the same term twice. Two variants ("running shoes for men | best men's running shoes") read as spam and consume the space a differentiator could use.
- Google rewrites titles often — roughly 60% of pages in Zyppy's post-2021 study. It rewrites most when the title is stuffed, duplicated across the site, or ignores the page's own H1. A concise, accurate title is how you keep control of your own snippet.
- Differentiators that earn clicks in a crowded SERP: year for time-sensitive queries, numbers, the qualifier that separates you (free, no signup, with data, for teams).
- Templated titles across thousands of pages are fine — as long as the variable part is genuinely different (`{product} {size} — {brand}`), never the same 8 words plus an ID.

## Meta Description

- 150-160 characters. Google rewrites the majority of them (roughly 60-70% per Ahrefs' study) — write them anyway: when yours is kept, it is because it matched the query.
- Its only job is CTR. It is not a ranking factor, and query terms in it get bolded, which is why the description should contain the phrasing users search.
- Never duplicate across pages; duplicates are the fastest way to get an auto-generated snippet instead.
- Write the promise, not a summary: what the reader gets and why this page over the four above it.
- Leave it out on pages where an auto-generated snippet from the content will match the query better than any static text — long reference pages and glossaries.

## Headings

- One H1, matching the page's subject in the user's language, not in your internal jargon.
- H1 and title tag should differ: the title targets the SERP, the H1 targets the reader on the page. That is two chances at phrasing, not a contradiction.
- H2/H3 hierarchy without skipping levels. Question-shaped H2s that mirror People Also Ask phrasing are what snippet extraction reads.
- Headings are structure, not keyword slots. Do not use H1 for the logo, and do not wrap navigation in headings.

## Body Copy Placement

- Answer the query in the first ~100 words; both featured snippets and AI summaries extract answer-first pages.
- Cover the term and its natural variants where they belong: opening, one or two subheads, image alt where accurate. No density threshold exists; stuffing detection is pattern-based.
- Entities beat synonyms: naming the specific tools, standards, places, and people the topic involves is what marks depth, not a list of related phrases.
- Anchors from within the body carry more weight than boilerplate nav links; place a link to the money page in prose where the reader is already convinced.

## URLs

- Short, lowercase, hyphenated, and stable: `/email-marketing/` beats `/page?id=123` and `/emailmarketing/`.
- Words in a URL are a weak signal but a strong usability and CTR one — the URL is displayed in the SERP and read out in link text.
- No dates in URLs for evergreen content; the date blocks refreshing and dates the page in the SERP.
- Match the folder to the topic cluster. `/guides/email-marketing/` communicates where the page belongs to Google and to the reader.
- Never change a URL without a 301, and never change one for cosmetics alone: the ranking risk exceeds the tidiness gain.

## Images

- Alt text describes the image for someone who cannot see it: "golden retriever catching a frisbee on grass", not "dog" and not "best dog training tips 2026".
- Descriptive file names with hyphens; the file name is a real signal in image search where alt is absent.
- Ship modern formats (WebP or AVIF) and correct dimensions. WebP is typically 25-35% smaller than JPEG at similar quality.
- Lazy-load below the fold only. Lazy-loading the hero image delays the LCP element and is a common self-inflicted vitals failure.
- Always set width and height (or aspect-ratio) so the layout does not shift while the image loads.
- Images that should rank need context: a caption, surrounding text, and an image sitemap entry on media-heavy sites.

## Internal Signals A Page Needs Before Publication

- At least two contextual internal links pointing in, from pages that already have traffic.
- One clear next action for the reader, linked in prose.
- Breadcrumbs reflecting the real hierarchy.
- Author byline linked to a bio page when the topic requires expertise.

## On-Page Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Rewriting titles sitewide in one release | You lose attribution for a change worth measuring | Batch by template, one batch per window |
| Optimizing a page with no impressions | Snippet work compounds impressions that do not exist | Fix indexing or intent first |
| Keyword-stuffed alt text | Zero ranking value, real accessibility damage, pattern-detectable | Describe the image; keywords only when they are the accurate description |
| Title matching the H1 exactly everywhere | Wastes the second phrasing opportunity | Title for the SERP, H1 for the page |
| Adding "2026" to titles without updating content | The snippet promises freshness the page does not deliver; CTR rises then bounces | Update the content, then the year |
| Deleting meta descriptions because "Google rewrites them" | You forfeit the cases where yours is kept — typically the exact-match queries | Write them for the pages that matter, skip them on the long tail |
