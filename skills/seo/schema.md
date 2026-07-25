# Structured Data — What Still Earns a Rich Result

Structured data does not raise rankings. It makes a page eligible for a visual treatment, and the treatment raises CTR. So the only question worth asking before writing markup is: **which rich result does this unlock, and does Google still show it?** Markup for a result Google retired is pure cost.

**Contents:** [Eligibility Reality Check](#eligibility-reality-check) · [Implementation Rules](#implementation-rules) · [Article](#article) · [Product](#product) · [LocalBusiness](#localbusiness) · [Video](#video) · [Structured Data Traps](#structured-data-traps)

## Eligibility Reality Check

| Type | Status for most sites | Worth implementing |
|---|---|---|
| Product / merchant listing | Active: price, availability, review stars, shipping and returns | Yes for ecommerce — the highest-value markup there is |
| Review snippet | Active, but self-serving reviews (your site reviewing its own products or business) are not eligible | Yes, for third-party review content |
| Breadcrumb | Active: replaces the URL line in the SERP | Yes, cheap and universal |
| Video | Active: thumbnails, key moments, video tab | Yes if video is the content |
| Event, Recipe, JobPosting, Course, Book | Active in their verticals | Yes if that is your business |
| Article / NewsArticle | No standalone rich result, but it feeds Top Stories and Discover eligibility signals | Yes for publishers |
| Organization / Person | No visual result; feeds entity understanding and knowledge panels | Yes, once, sitewide |
| LocalBusiness | Supports the local entity, not a SERP feature by itself | Yes for local |
| FAQPage | Since August 2023 shown only for well-known authoritative government and health sites | No, unless you are one |
| HowTo | Rich result deprecated in 2023 | No |
| Sitelinks searchbox | Feature removed by Google in late 2024 | No — delete the markup |
| Anything else | Not supported by Google as a rich result | Only if a non-Google consumer reads it |

Valid markup is not the same as a supported feature: schema.org has hundreds of types, and Google renders a short list.

## Implementation Rules

- JSON-LD in a `<script type="application/ld+json">` block. Head or body both work; JSON-LD is the format Google recommends and the only one you can change without touching templates.
- One entity per thing, connected by `@id` instead of repeated: an `Organization` block with a stable `@id`, referenced as `publisher` from every `Article`, is how a site describes itself once.
- Markup must match content visible on the page. Marked-up text a user cannot see is a spam violation, and it is the most commonly enforced structured-data rule.
- Required properties or nothing: a missing required property makes the item ineligible, not degraded. Recommended properties are warnings — they widen eligibility but do not block it.
- Injected by JavaScript works only if the render succeeds; server-rendered markup avoids a whole class of "why did the rich result disappear" incidents.
- Validate twice: the Rich Results Test for Google eligibility, the schema.org validator for syntax. GSC's Enhancements reports show what Google actually parsed on live URLs — the only source that reflects reality after deployment.

## Article

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "...",
  "author": {"@type": "Person", "name": "...", "url": "https://example.com/author/..."},
  "datePublished": "2026-01-15",
  "dateModified": "2026-03-02",
  "image": ["https://example.com/hero-16x9.jpg"],
  "publisher": {"@id": "https://example.com/#organization"}
}
```

- `author.url` pointing to a real bio page is the machine-readable half of E-E-A-T; a bare name string says nothing.
- `dateModified` must correspond to a real content change; bumping it without editing is deceptive and detectable.
- `headline` should match the visible H1, not the SEO title.

## Product

```json
{
  "@type": "Product",
  "name": "...",
  "sku": "...",
  "offers": {"@type": "Offer", "price": "49.00", "priceCurrency": "USD",
             "availability": "https://schema.org/InStock"}
}
```

- `offers` with price, currency, and availability is what produces the price and stock treatment; missing currency is the classic silent failure.
- `aggregateRating` requires real reviews visible on the page. Ratings assembled from nothing are the fastest route to a structured-data manual action.
- Variants: model the parent with `ProductGroup` and the buyable variants as `Product` items with `hasVariant`, rather than shipping near-identical Product blocks per color.
- Merchant listing experiences also read shipping and return policy properties — supplying them is what separates a listing that shows delivery information from one that does not.

## LocalBusiness

```json
{
  "@type": "Dentist",
  "name": "...",
  "address": {"@type": "PostalAddress", "streetAddress": "...", "addressLocality": "...",
              "postalCode": "...", "addressCountry": "..."},
  "telephone": "+1-555-0100",
  "openingHoursSpecification": [...],
  "sameAs": ["https://www.google.com/maps/place/...", "https://www.facebook.com/..."]
}
```

- Use the most specific subtype that exists (`Dentist`, `Restaurant`, `LegalService`), not the generic parent.
- `sameAs` linking to the Google Business Profile and major profiles is the cheapest entity-reconciliation signal available.
- Name, address, and phone must be byte-identical to the ones published elsewhere.

## Video

- `VideoObject` with `name`, `description`, `thumbnailUrl`, `uploadDate`, and `contentUrl` or `embedUrl`.
- `clip` or `SeekToAction` markup unlocks key moments — the multi-link treatment that dominates vertical space in a video result.
- The video must be the page's main content and must be crawlable; a video behind a script-only player is invisible.

## Structured Data Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Marking up content not visible on the page | Policy violation; triggers structured-data manual actions | Mark up what renders |
| Ratings without reviews, or self-serving reviews | Ineligible by policy and enforceable | Third-party reviews, real and displayed |
| FAQ markup on every page "for the CTR" | The rich result is gone for ordinary sites; you carry the maintenance for nothing | Keep FAQ content for users; drop the markup |
| Article and Product on the same page | Contradictory entity claims; Google picks one or neither | One primary entity per URL |
| Copying a competitor's JSON-LD | Their `@id`s, URLs, and organization data travel with it | Generate from your own data |
| Never checking the Enhancements report after launch | Templated errors multiply silently across thousands of URLs | Review the report a week after any template change |
| Stale `availability` and `price` | Mismatch with the page triggers loss of eligibility, and in shopping surfaces, suppression | Generate from the same source as the page |
