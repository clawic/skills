# International — Languages, Countries, and hreflang

Two different problems get called "international SEO": serving the same language to several countries, and serving several languages. The first is a duplication problem, the second is a coverage problem, and hreflang is how you tell Google which page belongs to whom.

## Choosing The Structure

| Structure | Example | Strengths | Costs |
|---|---|---|---|
| Subfolder | `example.com/de/` | Inherits domain authority, one property to manage, cheapest to run | Weaker local signal than a ccTLD |
| Subdomain | `de.example.com` | Separable hosting and stacks | Authority consolidation is less certain; more infrastructure |
| ccTLD | `example.de` | Strongest country signal, sometimes required by law or by trust | Each domain earns authority from zero; multiplies every technical task |
| Parameters | `example.com?lang=de` | None worth having | Fragile, poor for users and for crawlers |
| Anything else | — | — | Default to subfolders unless a ccTLD is a business or legal requirement |

Language and country are different axes. `/de/` (German language) and `/de-ch/` (German for Switzerland) answer different needs; do not create country variants you cannot keep genuinely distinct.

## hreflang Rules

- Every page in a set lists **all** versions including itself. Missing self-reference and missing return links are the two errors that make the whole cluster ineffective.
- Return links must be reciprocal: if A points to B, B must point to A, or Google ignores the pairing.
- Codes: ISO 639-1 for language, ISO 3166-1 Alpha-2 for region, in that order. `en-GB`, not `en-UK` (which is invalid and silently ignored). Language alone (`de`) is valid; region alone is not.
- `x-default` marks the fallback for users whose language and country match nothing — usually the global or language-selector page.
- Pick one implementation: HTML `<link>` tags in `<head>`, HTTP headers for non-HTML files, or XML sitemap entries. Sitemap implementation scales best for large sets and keeps templates clean; mixing methods creates contradictions.
- hreflang URLs must be absolute, canonical, indexable, and 200. Pointing hreflang at a redirect or a noindex page invalidates the annotation.
- hreflang and canonical must agree: each page self-canonicalizes and hreflang-references the others. Canonicalizing a translation to the English original deindexes the translation.
- hreflang is not a ranking booster. It swaps which of your pages shows in which market, and it prevents same-language duplicates from cannibalizing each other.

## Same Language, Many Countries

The most common real case (`en-US`, `en-GB`, `en-AU`) and the one where duplication bites:

- hreflang handles the swap, but pages that are byte-identical give Google little reason to keep all of them; expect one to dominate.
- Differentiate what a local buyer needs: currency and pricing, shipping and returns, phone and support hours, spelling, local case studies, legal and tax terms.
- If you cannot differentiate, publish one page for the language and let hreflang do currency detection client-side instead of spawning near-duplicate URLs.

## Geotargeting And Detection

- Search Console international targeting (for subfolders and subdomains) sets a country preference; ccTLDs are targeted automatically and cannot be changed.
- Never auto-redirect by IP. Crawlers fetch mostly from US IPs and will only ever see one version, and travelers get the wrong site. Offer a dismissible banner with a link instead, and let the URL the user chose stick.
- Server-side language negotiation on the root URL is acceptable when each language also has its own stable, linkable URL.
- Hosting location is a weak signal at best; a CDN is a better answer to latency than a local server is to ranking.

## Translation Quality

- Machine translation without human review reads as low-value content in the target market, and it is what scaled-content policies describe.
- Translate the intent, not the string: keyword research must be redone per market. The literal translation of a head term is frequently not what locals search, and search volume distributions differ by market.
- Localize URLs, titles, metadata, structured data, images with text, dates, units, and currency — a page in German with English URLs and dollar prices is half-translated.
- Local search engines matter where they have share (Yandex in Russia, Naver in South Korea, Baidu in mainland China); their requirements differ enough that treating them as "Google with a different name" fails.

## Diagnosing International Problems

| Symptom | Likely cause |
|---|---|
| Wrong country version ranks in a market | Missing or broken hreflang return links; or that version is simply the stronger page |
| Translations not indexed | Canonical points to the source language, or the pages are near-identical after machine translation |
| Traffic from one market only | No hreflang, no localized content, no local links |
| GSC reports "no return tags" | The other side's annotation is missing, on a redirect, or uses a different URL form (trailing slash, www) |
| Duplicate content across English variants | Identical pages; differentiate or consolidate |
| Anything else | Fetch each version as Googlebot and compare the head block URL by URL |

## International Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| IP-based auto-redirects | The crawler sees one version; users get trapped | Banner with a link, user choice persisted |
| `en-UK` | Not a valid region code; the annotation is dropped | `en-GB` |
| hreflang pointing at redirects | Annotations require the final canonical URL | Point at 200-status canonicals |
| Machine-translating the whole site at once | Scaled low-value content in every market simultaneously | Translate the pages with demand, reviewed by a human |
| One ccTLD per country from day one | Every domain restarts authority; the workload multiplies by the number of markets | Subfolders until a market proves itself |
| Copying keyword lists across markets | Search language is not a translation | Redo research per market |
