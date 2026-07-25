# Publishers — Top Stories, Google News, and Discover

Publisher SEO runs on different clocks. Evergreen SEO compounds over months; news visibility is decided in minutes and decays in hours, and Discover ignores queries entirely. Treating them as one channel produces a newsroom optimized for neither.

## The Three Surfaces

| Surface | Selection | Traffic shape | What moves it |
|---|---|---|---|
| Top Stories | Freshness, entity relevance, publisher authority for the story | Minutes-long spikes on breaking stories | Speed of publication, indexing latency, being an authoritative source on that entity |
| Google News (tab and app) | Publisher-level interest and topic following | Steady, topic-driven | Consistent coverage of a beat, clean structure, publisher entity signals |
| Discover | Interest-based feed, no query at all | Enormous, volatile, unpredictable spikes | Topic interest, images, headline appeal, entity coverage, freshness |
| Classic search | Query intent | Compounding | Everything in the rest of this skill |

Inclusion in Google News no longer requires an application — eligible sites are considered automatically; there is a Publisher Center for managing publication details and organization data, not a gate to entry. AMP has not been required for Top Stories since the 2021 page experience update; speed still matters, the format does not.

## Being Indexed Fast Enough To Matter

- A story indexed twenty minutes late has already lost the Top Stories window on a breaking event.
- Ship a news sitemap containing only the last 48 hours of articles, updated on publish.
- Link new articles from the homepage and from the relevant section hub the moment they publish; those are the pages Google crawls most often.
- Keep the article URL stable from the first publish: URL rewrites after a story goes live break early links and reset discovery.
- Publish the first correct version rather than a placeholder; a stub that gets substantially rewritten competes against its own earlier indexing.
- Watch Crawl stats for latency: a site whose crawl rate collapses under traffic spikes loses exactly when it matters.

## Article Construction

- Headline in the H1 that a human would click, and a title tag that can differ from it — the SEO title can carry the entity and the event, the on-page headline can carry the drama.
- Lead with the news: what happened, who, when, where, in the first paragraph. Both extraction and reader retention depend on it.
- `NewsArticle` markup with accurate `datePublished` and `dateModified`, author with a linked bio, and publisher organization.
- Author pages with real credentials and history are load-bearing for news topics; anonymous bylines undercut every quality signal.
- Update in place for developing stories and mark the update visibly; spawn a new article only when the story is genuinely a new event.
- Link to the primary source and to your own prior coverage. Ongoing stories are entity clusters; the site that owns the cluster wins the follow-up queries.

## Discover

- No query, no keyword targeting, no reliable ranking to track. Traffic arrives or does not; a page can get more traffic in six hours than in the previous six months.
- Eligibility follows normal indexing plus interest. Large, high-quality images (Google's guidance is 1200px or wider) with `max-image-preview:large` are the mechanical prerequisite most sites miss.
- Headlines must be accurate and interesting at once. Clickbait and exaggerated headlines are named in Google's Discover guidance as a reason for exclusion.
- Content that performs: timely, entity-driven, human-interest, product and lifestyle pieces with a strong image. Reference and evergreen content rarely appears.
- Volatility is structural. Never build a revenue forecast on Discover; treat it as upside and keep search as the base.
- The Discover report in Search Console appears only once you receive Discover traffic, and it is the only place to measure it.

## Paywalls And Subscriptions

- Use structured data for paywalled content (`isAccessibleForFree: false` plus a `cssSelector` marking the paid section) so Google can index the full text without treating it as cloaking.
- Any other approach that shows Googlebot more than a user gets is cloaking, and it is enforced.
- Metered access is compatible with indexing; be consistent about what the meter allows.

## Archives And Evergreen On A News Site

- Archives accumulate into the site's quality average: thousands of ten-year-old thin wire stories drag sitewide assessments.
- Prune deliberately: consolidate topic archives, noindex tag pages with almost nothing on them, redirect or 410 abandoned sections.
- Convert recurring events into durable hubs updated each cycle ("election results", "the annual awards") rather than a new orphan page per year.
- Evergreen explainers are what pay between news cycles — they rank on the classic clock and feed the news cluster with internal links.

## Publisher Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Rewriting URLs after publish | Breaks early links and restarts discovery | Fix the slug before the first publish |
| Republishing old articles with today's date | Deceptive freshness; detectable | Update content substantively or leave the date |
| Chasing Discover with clickbait headlines | Named as an exclusion reason in Google's own guidance | Accurate headlines with a genuine hook |
| Treating Discover traffic as forecastable | It disappears without notice | Base plans on search, count Discover as upside |
| Publishing a new page per update to a live story | Splits signals across the same event | Update in place, timestamp the change |
| Letting the archive rot | Sitewide quality reassessments read the whole domain | Scheduled pruning and consolidation |
| Blocking crawlers to protect content while wanting Top Stories | You cannot be selected from content that is not readable | Use paywall structured data instead |
