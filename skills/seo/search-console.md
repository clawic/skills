# Search Console — Reading The Only First-Party Data You Get

Every other SEO number is modeled. Search Console is measured. Knowing its quirks is what separates a diagnosis from a story.

## Property Setup

- Domain property (DNS verification) aggregates every subdomain and protocol; URL-prefix properties isolate one. Have both: the domain property for totals, prefix properties for a section you want to watch on its own.
- Add a prefix property per major section (`/blog/`, `/docs/`, `/es/`) when the site is large — it makes per-section indexing and query reporting trivial without exports.
- Give the client's developer and the analyst read-only access on day one; audits stall waiting for it.

## Quirks That Break Analyses

- **16 months** of history, and no more. Export monthly if year-over-year matters; the day you need month 17 is the day you cannot have it.
- The UI table caps at 1,000 rows; the API returns up to 25,000 rows per request. Any site above trivial size needs the API, Looker Studio, or the bulk BigQuery export for real work.
- **Anonymized queries**: rare queries are withheld, so the sum of query rows is always less than the totals row. A "missing clicks" investigation usually ends here.
- **Average position** is the average of the best position per impression, weighted by impressions. Ranking a new page at 60 for a thousand new queries drags the average down while traffic grows. Never report it as a success metric.
- Filtering by query changes the page list to pages that appeared for that query — this is the cannibalization view, not a page report.
- Data lags roughly two to three days; the last days of any chart are incomplete.
- Discover and News have separate reports with their own quirks and do not appear in the Search report.
- Search Appearance filters exist for some features, but there is no filter for AI Overview presence.

## The Five Reports Worth Reading

| Report | Question it answers | Trap |
|---|---|---|
| Performance → Search results | What already earns impressions and clicks | Comparing periods of different lengths, or including branded queries in "SEO growth" |
| Page indexing | What Google chose to index and why not | Treating "Discovered — currently not indexed" as a bug rather than a demand signal |
| URL Inspection (live test) | What Google fetches and renders right now | Reading the indexed version and thinking it is the current page |
| Crawl stats (Settings) | Whether the host is healthy under crawl | Ignoring a rising 5xx or a collapsing crawl rate |
| Links | Internal and external link counts | Using it as a full backlink index; it undercounts and does not score |

## Workflows That Produce Decisions

**Striking distance** — Performance → last 3 months → filter position 4-15, impressions at or above `min_impressions` (SKILL.md Configuration; default 100/month, raise it on large sites). Sort by impressions. These pages already have relevance; they lack CTR or authority (SKILL.md rule 3).

**Snippet problems** — For each important page, compare its CTR to your own site's average CTR at that position. Far below your baseline and the snippet or the SERP layout is the problem, not the ranking.

**Cannibalization** — Filter by query, open the Pages tab. Two URLs trading impressions across weeks is the signature.

**Content decay** — Compare the last 3 months against the same 3 months last year, by page. Pages down 30%+ year over year with unchanged position are losing to freshness or to SERP layout; pages down in position lost to a competitor. Different fixes.

**Query gaps** — Filter one page's queries: everything it earns impressions for but does not cover in the content is the next revision's outline.

**Regex filters** — Question queries (`^(how|what|why|when|where|who)`), brand vs non-brand (`^(?!.*brandname)`), and long-tail by word count. Brand exclusion is the single most valuable filter, because branded queries hide whether SEO is actually growing.

## Reporting To Stakeholders

- Lead with clicks, conversions, and revenue from organic; positions are diagnostics, not outcomes.
- Split branded and non-branded in every report. Growth that is entirely branded came from marketing, not from SEO — say so.
- Compare like periods: 28 days against the previous 28 and against the same 28 last year. Month-over-month on a seasonal business is a fiction generator.
- Annotate the timeline with what shipped and with Google's announced updates; a chart without annotations invites the wrong causality.
- Include a "what we expect next" line with the mechanism and range from SKILL.md's What Takes How Long, never a promised date.

## Proving A Change Worked

SEO has no built-in control group, so causality has to be constructed:

- **Control group test**: split a template's URLs into a test set and a matched control set of similar pages, ship the change to the test set only, and compare clicks per page against the control over 4-8 weeks. This is the only method that survives an algorithm update landing mid-test.
- **Pre/post without a control** answers nothing on its own: seasonality, updates, and competitors move the same numbers.
- Underpowered by default: a handful of pages with a few hundred clicks cannot detect a 5% effect. Test on templates with many similar URLs, or accept that you are making a judgment call and say so.
- Change one variable per window, and log the date. An unannotated timeline is where attribution goes to die.
- Watch for the pages excluded from your own test: if the control group also improves, you measured the season.

## Forecasting Honestly

`expected clicks = search volume × CTR at the realistic position × (1 − feature discount)`

- Use your own CTR curve from GSC — filter your queries at each position band and average — before reaching for any published curve. Site and vertical differences are large.
- State the assumption set with every forecast: target position, timeline, and what must ship. A forecast without conditions becomes a promise.
- Give a range and a confidence, and revisit at the halfway mark. Being wrong loudly and early costs less than a precise number nobody revisits.

## Search Console Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Reporting average position gains | Query mix moves it independently of performance | Report clicks and per-query positions |
| Comparing to yesterday | Data is incomplete for the last few days | Use 28-day windows |
| Using `site:` instead of the indexing report | The operator is an unreliable estimate | Page indexing + URL Inspection |
| Trusting the 1,000-row table | Truncation hides the long tail that is most of the traffic | API, BigQuery export, or Looker Studio |
| Mixing branded and non-branded | Hides whether SEO is working | Regex-exclude the brand |
| Deleting the property after a migration | You lose the old URL history you will need | Keep both properties; keep exporting |
