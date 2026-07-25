# AI Search — Overviews, Assistants, and Citations

Two things changed at once: Google answers more queries on the results page, and a share of research now starts in an assistant instead of a search box. Both reward the same underlying asset — a page that is easy to extract a correct, attributable answer from — and both punish the same behavior: writing for a click that the surface will no longer produce.

## What The Numbers Support

- Clicks fall on queries where a summary answers the question. Pew's 2025 browsing study found users clicked a result on 8% of visits with an AI summary versus 15% without — a large relative drop on informational queries, measured on a browsing panel rather than on your site.
- Citation and classic ranking overlap substantially but far from completely; published 2024-25 analyses put the overlap between AI Overview sources and the classic top 10 roughly in the 40-60% band depending on vertical and method. Ranking well is the best available lever, not a guarantee.
- Assistant referral traffic is small in absolute terms for most sites and converts differently: fewer sessions, higher intent, usually undercounted because some assistants strip referrers.
- Do not quote a single industry percentage as fact to a client. Measure your own: compare clicks against impressions per query group over the last year.

## The Two Different Games

| Surface | How it selects | What to optimize |
|---|---|---|
| Google AI Overviews and AI Mode | Grounded in Google's index; sources skew toward pages already ranking and toward pages with a clean, extractable answer | Classic ranking, plus answer-first structure and unambiguous facts |
| Assistants that browse live (ChatGPT, Perplexity, Copilot) | Retrieval plus live fetch; heavily influenced by what their crawlers can read | Crawlability for their agents, clear factual pages, third-party mentions |
| Assistants answering from training data | Frozen corpus, no fetch | Nothing tactical — only long-run presence and being written about elsewhere |
| Anything else | Assume grounded retrieval | Serve clean HTML and unambiguous facts |

## What Earns Citations

- **Answer-first blocks**: the direct answer in the first 100 words or under the exact question heading, self-contained enough to quote without the surrounding page.
- **Facts a model can lift with attribution**: named numbers, dates, definitions, comparisons in a table. Vague prose gets summarized away; a table row gets cited.
- **Stable, current pages**: contradictory or undated claims lose to a page that states its date and its scope.
- **Being the primary source**: original data, first-hand tests, and public statistics get cited by both machines and journalists. Aggregations of other people's work are exactly what summaries replace.
- **Third-party consensus**: assistants weight what other sites say about you. Listicles, review sites, forums, and documentation that mention you shape what gets recommended when your brand is not the query.
- **Clean HTML**: content behind interaction, heavy client-side rendering, or aggressive bot blocking simply is not read.

## Crawler Policy Is Now A Business Decision

- Google's AI features are served from the standard Google crawl. Blocking `Google-Extended` withholds content from certain Gemini uses but does not remove you from AI Overviews, which are grounded in the search index. There is no "search yes, AI Overview no" switch.
- Non-Google AI crawlers are separate user agents and can be allowed or blocked individually in robots.txt. Blocking them removes you from those assistants' answers and from any referral they would send.
- Decide deliberately per business model: a publisher monetizing pageviews and a SaaS company that wants to be recommended have opposite interests. Record the choice with its reason; this is the kind of decision that gets reversed by a stranger six months later.
- `llms.txt` is a proposed convention, not a standard any major engine has confirmed reading. Cost is near zero, benefit is unproven — publish it if you like, and never present it as an optimization.

## Measuring

- No public tool reports AI Overview presence reliably at scale. Sample manually: take your top 50 queries, search them in the target market, and record which show an AI Overview and whether you are cited.
- Watch for the fingerprint in Search Console: stable or rising impressions with falling CTR at unchanged position on informational queries.
- Segment queries by type. Informational and definitional queries lose the most clicks; navigational, transactional, and local queries are far less affected — which is a strategy instruction, not just a report.
- Track referrals from assistant domains in analytics as their own channel, and expect the number to understate reality.
- Brand mentions without links matter more in this environment than in the last one; monitor them alongside links.

## Strategy Shift That Follows

1. Rebalance toward queries a summary cannot satisfy: transactional, comparative with personal fit, local, tool-driven, and anything requiring your data or account.
2. Keep informational content that earns citations and trust, but stop measuring it in raw sessions alone; its job includes being the source that gets named.
3. Invest in the assets a model cannot synthesize: proprietary data, tools and calculators, community, and named expertise.
4. Strengthen brand search. When an assistant recommends categories rather than URLs, being the name people then search is the durable position.

## AI Search Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Writing pages "for LLMs" in a special format | No engine documents a special format; you get a worse page for humans | Answer-first structure, clean HTML, real facts |
| Blocking every AI crawler by default | Removes you from assistant answers and their referrals with no ranking benefit | Decide per crawler, per business model |
| Assuming AI Overviews killed the click | Effects vary hugely by query type; category-wide claims mislead planning | Measure your own CTR by query segment |
| Chasing citation on queries you cannot monetize | Citation without clicks is a vanity metric | Prioritize queries where the click is the transaction |
| Publishing an `llms.txt` and reporting it as optimization work | No engine has confirmed reading it | Publish it if you like, and count it as zero |
| Stuffing "as an expert" phrasing to sound authoritative | Models weight corroboration, not adjectives | Get cited elsewhere; publish verifiable evidence |
