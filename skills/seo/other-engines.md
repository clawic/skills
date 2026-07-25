# Other Engines — Bing, IndexNow, and Regional Search

Google is most of the traffic in most markets, and it is not all of it. Bing's index feeds Copilot and several assistants; regional engines dominate whole countries. The work is small and mostly one-time, which is exactly why nobody does it.

## Why Bing Is Worth An Afternoon

- Bing Webmaster Tools is free, imports the site from Search Console in a couple of clicks, and reports crawl, index, and query data with fewer restrictions than GSC.
- Its index grounds Copilot and, at times, other assistant products. Being absent from Bing can mean being absent from answers that never touch Google.
- Its ranking preferences are less link-dependent and more literal about on-page relevance: exact-term usage in titles and headings, clean HTML, and older domains do relatively better than they do on Google.
- Social and brand signals are described as part of Bing's picture in its own guidance; Google denies the same.
- The URL and Content Submission APIs let you push URLs directly instead of waiting to be discovered.

## IndexNow

- A push protocol supported by Bing, Yandex, Seznam, and others: you notify the engines when a URL is created, updated, or deleted, instead of waiting for the next crawl.
- Implementation is a key file at the site root plus one HTTP request per changed URL; most major CMS platforms and CDNs have a plugin or a toggle.
- Google has not adopted it, so it changes nothing for Google traffic. Treat it as a cheap improvement for the engines that do use it, especially for news, ecommerce stock changes, and large catalogs.
- Do not spam it: submitting unchanged URLs repeatedly gets the key throttled.

## Regional Engines

| Market | Engine | What differs |
|---|---|---|
| Russia | Yandex | Its own webmaster tools and index; behavioral signals weigh heavily; Cyrillic content and local hosting matter |
| South Korea | Naver | A portal, not a web index: blogs, cafés, and knowledge-in sections rank ahead of external sites; presence on Naver's own properties is the strategy |
| China | Baidu | Requires local hosting and an ICP license for realistic performance; simplified Chinese; Google services on the page break rendering |
| Japan | Google, with Yahoo! Japan on Google's index | Work Google, but respect local formatting and mobile conventions |
| Privacy engines (DuckDuckGo, Ecosia, Brave) | Mostly syndicated from Bing or their own crawl | Bing coverage usually covers you |
| Anywhere else | Google | Check actual market share before spending anything |

Never assume a Google playbook transfers. Where a portal owns the market, the winning move is publishing on the portal's properties, not out-ranking them from outside.

## Apple, App, And In-Product Search

- Apple's Spotlight and Siri suggestions crawl the web with Applebot; blocking it removes you from those surfaces. Apple Business Connect is the local profile equivalent for Apple Maps.
- Marketplaces (Amazon, Etsy, YouTube, app stores) are ranked catalogs with their own algorithms. They are outside this skill, but they compete for the same query, and a query dominated by a marketplace is often a query to win inside it rather than on the open web.

## What To Actually Do

1. Verify the site in Bing Webmaster Tools and import from Search Console.
2. Submit the sitemap and check its index coverage once a quarter — an entire site missing from Bing is usually one blocked user agent.
3. Enable IndexNow if the platform makes it a toggle.
4. Confirm your robots.txt does not block `bingbot`, `Applebot`, or the regional crawlers you care about; blanket blocks written for scrapers routinely catch them.
5. Only if a regional market is a business priority: verify with that engine's tools, and get local advice — hosting, licensing, and content norms are not translatable.
