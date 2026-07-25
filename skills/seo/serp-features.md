# SERP Features — Snippets, Packs, and Pixel Position

Rank is a coordinate; visibility is a pixel measurement. On a modern results page, organic result #1 can sit below an AI summary, four ads, a shopping row, and a People Also Ask block. Optimizing features is how you take back vertical space you cannot win by ranking.

## Read The SERP Before Deciding Anything

For a target query, record: which features appear, in what order, how far down the first organic result starts, and who owns each feature. That layout is the ceiling on what any ranking is worth (SKILL.md rule 9 turns it into a click estimate).

| Feature | Who wins it | Practical lever |
|---|---|---|
| Featured snippet | Usually a page already in the top 10 with a directly extractable answer | Format the answer to match the snippet type |
| People Also Ask | Pages answering the exact question phrasing | Question-shaped H2 plus a 40-60 word answer beneath it |
| Sitelinks | Chosen algorithmically from site structure | Clear architecture, unambiguous titles, real navigation |
| Image pack | Image search relevance, page context | Descriptive file names and alt text, image sitemap, context around the image |
| Video results / key moments | Video-primary pages and YouTube | VideoObject markup, chapters, transcript on the page |
| Top Stories | News-eligible publishers | Publisher-side eligibility, freshness, entity coverage |
| Local pack | Proximity, relevance, prominence | Google Business Profile work; organic changes do not move it |
| Shopping / merchant listings | Feed plus product markup | Feed accuracy and identifiers |
| Reviews stars | Eligible review markup on non-self-serving content | Real third-party review data |
| AI Overview | Grounded selection across the index | Answer-first, extractable, current pages |
| Anything else | — | Check whether the feature is even available in the target market |

## Featured Snippets

- Snippet types map to answer format: paragraph (definition or direct answer, roughly 40-60 words), list (steps or ranked items under an ordered or unordered list), table (comparisons and specs), video (procedural queries).
- Match the type the SERP already shows; Google rarely swaps a table snippet for a paragraph because you preferred prose.
- Place the answer immediately under a heading that matches the query phrasing, self-contained, no pronouns pointing at earlier text.
- Snippets are usually taken from pages in the top ten, most often not from position 1 — a page at 4-8 with a better-formatted answer routinely takes it. That makes snippets the cheapest visibility upgrade on striking-distance pages.
- Winning a snippet can reduce clicks when the answer fully satisfies the query. Take the snippet when it also advertises depth (a step list that requires the page), skip the fight when the answer is a single number a searcher will not follow up on.
- Losing one is normal: they rotate. Chase the format, not the day-to-day.

## People Also Ask

- Each PAA question is a real query with its own answer source. Answering three or four of them properly on one page is a cheap way to hold extra space.
- Expand PAA boxes repeatedly to harvest the question tree — it is free keyword research on the exact phrasing users type.
- Answer format: question as H2 or H3, direct answer in the first sentence, elaboration after.
- Do not build a page per PAA question. Cluster them into the page that owns the intent.

## Sitelinks, Brand, And The Knowledge Panel

- Sitelinks come from architecture and from user behavior; you influence them with a clean, shallow structure and distinct, descriptive titles, not with markup.
- The sitelinks search box was removed by Google in late 2024 — its markup does nothing now.
- A knowledge panel for an organization comes from entity understanding: consistent naming, `Organization` markup with `sameAs`, an authoritative "about" page, Wikipedia/Wikidata presence where warranted, and coverage elsewhere.
- Brand queries are the highest-converting traffic on most sites and the least defended. Verify that your own name returns your site, your sitelinks, and no competitor ads dominating the fold before optimizing anything else.

## Images And Video

- Image traffic is real for product, recipe, design, and travel topics: name files descriptively, write accurate alt text, keep images near the text that explains them, and ship an image sitemap on media-heavy sites.
- Google can show images from a page in the main results and in the image tab; both need the image to be crawlable and not lazily loaded past the point of render.
- For video: host the page as the canonical home of the video, add `VideoObject`, provide chapters and a transcript. Key moments occupy several lines of a result.
- YouTube and Google Search are different systems with different rankings; a video can dominate one and be invisible in the other.

## When You Cannot Win The SERP Yourself

Some queries are structurally closed: page 1 is review media, forums, and marketplaces, and no amount of quality puts a vendor's own page there. The play is to appear inside the results that already rank.

- Get included in the listicles and review sites that own the query: they take submissions, run comparisons, and update annually.
- Be present where discussion happens for your category — participating honestly and disclosing who you are. Undisclosed vendor posting gets removed by moderators and earns nothing.
- Marketplace and directory listings can be the ranking asset when the marketplace owns the SERP.
- Video results and platform-native content occupy positions your site cannot.
- Track these placements like rankings: which third-party page ranks, where you sit inside it, and when it was last updated. Getting moved from #7 to #2 inside somebody else's list is a real win with a real traffic effect.

## Measuring Feature Impact

- Search Console does not label features. The fingerprint of losing space is stable impressions and position with falling CTR.
- Compare your CTR at a position against your own site's average CTR at that position: a page far below your baseline is being crowded out or has a weak snippet.
- Sample the live SERP on mobile — the feature stack is far taller there, and the mobile layout is what most users see.

## SERP Feature Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Reporting rank without the layout | #1 below a summary and four ads is not what the client hears | Report pixel position and CTR |
| Building FAQ markup to win PAA | PAA selection is not driven by markup, and the FAQ rich result is gone for most sites | Answer the questions in the content |
| Chasing every snippet | Some snippets end the journey and cost you the click | Take snippets that advertise depth |
| One thin page per PAA question | Cannibalization plus thin content | Cluster into the intent's page |
| Ignoring brand SERPs | Competitors bid and rank on your name while you optimize category terms | Audit the brand SERP first |
| Assuming feature parity across markets | Availability differs by country and language | Check the SERP in the target market |
