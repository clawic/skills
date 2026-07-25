# Local — Map Pack, Profiles, and Multi-Location

Local search is a separate ranking system sitting on top of organic. The pack has its own inputs, its own spam problems, and one factor you can never change: where the searcher is standing.

## How The Local Pack Ranks

Three factors, per Google: **proximity** (the searcher's location — outside your control), **relevance** (categories, profile completeness, site content), **prominence** (reviews, links, citations, offline notability). Everything actionable moves relevance or prominence. Never promise pack rankings to a client for searches happening across town: proximity caps the ceiling, and rank varies block by block — which is why a single "we rank #2 locally" number is meaningless without saying from where.

## Google Business Profile

- Claim and verify. Unverified profiles do not rank, and verification is now the slowest step in most local projects — start it first.
- **Primary category is the strongest lever you control.** Pick the most specific match ("Personal Injury Attorney", not "Lawyer"); add secondary categories for genuinely offered services. Research it by checking the categories of the businesses already in the pack for your target query.
- Complete every field: hours (including holiday hours), services with descriptions, products, attributes, opening date, and photos. Google reports that businesses with photos get 42% more direction requests.
- Never stuff keywords into the business name. It violates the guidelines, competitors report it, and suspensions follow. The legal, real-world name only.
- Service-area business with no walk-in customers: hide the address and set service areas. A visible home address invites suspension and does not help.
- Post regularly, and seed the Q&A section with real customer questions answered by the business — if you leave it empty, competitors and spammers fill it.
- Keep the website link pointing at the specific location page, not the homepage, on multi-location profiles.
- Track profile changes: Google accepts edits from the public, and a competitor or a well-meaning user can change your hours or category.

## Suspensions And Spam

- Common triggers: keyword-stuffed name, address that is a virtual office or a mailbox, address shown for a service-area business, big edits in one session (name plus address plus category at once), category far outside the real business, and multiple profiles at one address.
- Reinstatement: gather evidence before appealing — business license, utility bill at the address, signage photos, and any registration documents. One well-documented appeal beats five thin ones.
- Make risky edits one at a time, spaced out, on an established profile.
- Competitor spam (fake locations, stuffed names) is reported through Google's business redressal process; document with screenshots. Enforcement is inconsistent — plan around it rather than depending on it.

## NAP Consistency

- Name, Address, Phone identical across the profile, the site, and the major directories: Yelp, Facebook, Apple Business Connect, Bing Places, and the industry directories that matter in your vertical.
- Google normalizes minor formatting ("St." vs "Street"); the real killers are old addresses and dead phone numbers left behind after a move, not punctuation.
- Local phone number over toll-free — toll-free reads as non-local.
- After a move: update the profile first, then the site, then the top directories. Stale citations suppress local rankings for months.
- Embed NAP in the site's markup, in text (not in an image), on every location page and in the footer.

## Reviews

- Rating, quantity, recency, and your response rate all feed prominence — and reviews are also the single biggest conversion factor in the pack.
- Respond to every review, negatives especially. The response is written for the next customer reading it, not for the reviewer.
- Steady velocity beats bursts. A sudden spike after years of silence looks purchased, because usually it is.
- Ask at the point of satisfaction with a direct review link. Never gate reviews (filtering so only happy customers are asked) — gating violates Google's policy.
- Never buy reviews: pattern detection leads to review removal or profile suspension, and removal takes the legitimate ones with it.
- Reviews mentioning the service and the city are the ones that read as relevance signals; asking "what did we do for you?" produces them naturally.
- Fake negative reviews can be reported, but the durable answer is volume of real ones.

## Citations And Local Links

- Structured citations: business directories carrying NAP. Get the core set right, then stop — the tenth directory adds nothing.
- Industry-specific directories (lawyers on Avvo, restaurants on TripAdvisor, trades on their trade bodies) outweigh generic ones.
- Data aggregators push NAP to many directories at once; fix the source rather than fifty listings.
- Unstructured mentions — local news, community blogs, event pages, sponsorships, chambers of commerce — double as local link building and are the strongest prominence lever available to a small business.

## Location Pages

- One page per real location, and one page per service × city actually served. Each must have unique substance: staff, photos, directions, parking, local case studies, service specifics, embedded map of that profile.
- Templated city-swap pages get filtered as doorway pages, and pages for cities you do not serve are an explicit policy violation.
- The page a profile links to should match the profile exactly: same NAP, same hours, same services.
- `LocalBusiness` markup on each location page with the most specific subtype and `sameAs` pointing at the profile.
- Multi-location sites need a store locator that is crawlable: real links to each location page, not a search box.

## Multi-Location Operations

- One profile per physical location, one owner account, consistent naming conventions across all of them.
- Bulk management through the profile's location groups; spreadsheets drift from reality within a quarter.
- Central branding, local specifics: identical text on 40 location pages is thin content 40 times over.
- Report per location — an average across locations hides the three that are suspended and the one carrying the brand.
- Franchise setups need explicit rules on who owns the profile, because a departing franchisee with the login is a business risk, not just an SEO one.

## Measuring Local

- Pack rank depends on the searcher's position, so measure on a geographic grid around the business rather than from one point.
- Profile insights show discovery vs direct searches, calls, direction requests, and website clicks — the closest thing to local conversion data.
- Call tracking: use a tracking number only where you can keep the real NAP number as the primary on the profile and the site, or use dynamic insertion that leaves the source number in the HTML.
- Organic and pack rankings move independently; report them separately or the story makes no sense.

## Local Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Keywords in the business name | Guideline violation, competitor reports, suspension | Real name; put keywords in categories and content |
| Promising pack rankings citywide | Proximity caps the radius | Show a grid, set expectations by distance |
| Buying or gating reviews | Removal or suspension, and the legitimate reviews go too | Ask everyone, respond to all |
| Fifty citations before fixing the profile | The profile is most of the ranking; directories are the tail | Profile, then core citations, then local links |
| City pages for cities you do not serve | Doorway pages | Only real service areas, with real local content |
| Changing name, address, and category in one session | Triggers reverification and suspensions | One change at a time |
| Homepage as the profile's website link on multi-location | Sends every location to the same generic page | Link the matching location page |
