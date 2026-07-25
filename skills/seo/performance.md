# Performance — Core Web Vitals Without the Cargo Cult

Speed is a tiebreaker in ranking and a multiplier on conversion. Treat it as the second: a fix that wins 0.4s of LCP pays in revenue whether or not it moves a position. Chasing a Lighthouse score of 100 pays nothing.

## The Three Metrics

| Metric | Good (p75, field) | What it measures | Usual cause when it fails |
|---|---|---|---|
| LCP | < 2.5s | Time until the largest visible element renders | Slow server, hero image not prioritized, render-blocking CSS/JS, client-side rendering |
| INP | < 200ms | Worst-case responsiveness to user interactions across the visit | Long JavaScript tasks, hydration, heavy third-party tags |
| CLS | < 0.1 | Unexpected layout movement | Images without dimensions, injected banners and ads, late-loading fonts |

INP replaced FID in March 2024 and is strictly harder: FID measured only the first interaction's input delay, INP measures the full interaction-to-paint of the worst interaction. Sites that passed FID trivially can fail INP.

## Field vs Lab

- **Field (CrUX)**: real users, 28-day rolling window, the data Google uses. This is why a deployed fix takes about four weeks to show up, and why a URL with little traffic falls back to origin-level data.
- **Lab (Lighthouse, PageSpeed's simulated run)**: one throttled load in a synthetic environment. Useful for diagnosis and for pre-launch comparison; it cannot pass or fail you.
- PageSpeed Insights shows both — read the field section first, and only debug in the lab section.
- Mobile field data is the one that matters; the desktop panel is often green while the mobile panel fails.
- No field data at all means low traffic, not good performance.

## Debugging LCP

Split the 2.5s budget into its four parts and find the one that owns the time: **TTFB → resource load delay → resource load time → element render delay** (web.dev's recommended split targets roughly 40% / 10% / 40% / 10%).

- TTFB over ~800ms: server, database, or origin distance. Cache the HTML, put a CDN in front, and stop rendering pages per request that could be static.
- Load delay: the browser learned about the LCP image late. Preload it (`<link rel="preload" as="image">`, or `fetchpriority="high"`), and never `loading="lazy"` on the hero — the single most common self-inflicted LCP failure.
- Load time: the image is too big. Correct dimensions, AVIF/WebP, responsive `srcset`.
- Render delay: render-blocking CSS or JS, or a client-side framework that paints after hydration. Inline critical CSS, defer the rest, and server-render the above-the-fold content.
- Carousels and video heroes: the LCP element becomes whatever renders last. Give the first slide a static, preloaded image.

## Debugging INP

- Find the interactions that hurt: the field diagnostics in PSI name the element; the Long Animation Frames data in DevTools names the script.
- Break up long tasks: work over 50ms blocks the main thread. Yield between chunks, move computation to a worker, and defer anything not needed for the first interaction.
- Hydration is the usual suspect on framework sites: the page paints, the user clicks, and nothing happens for a second because the bundle is still attaching handlers. Ship less JS to the client and hydrate progressively.
- Third-party tags (chat, A/B testing, tag managers loading tags loading tags) commonly own most of the main-thread time. Audit what the tag manager actually injects; the list is always longer than the marketing team believes.
- Animate with transform and opacity only; layout-triggering animations turn every interaction into a reflow.

## Debugging CLS

- Set explicit `width`/`height` or `aspect-ratio` on every image, iframe, embed, and ad slot.
- Reserve space for anything injected: cookie banners, promo bars, "you might also like" widgets. If it appears after paint, it must appear in space already allocated.
- Fonts: `font-display: swap` plus `preload` of the primary font, and a fallback with matched metrics to avoid a reflow when the webfont arrives.
- CLS is measured across the whole visit, not just load: content shifting during infinite scroll and on interaction counts.
- Verify with the field data, not by eye — the shifts that hurt are on devices slower than yours.

## Ordering The Work

`value = users affected × time saved × conversion sensitivity of the page`

Fix the templates carrying commercial traffic first, in this order, because each step removes the cause of the next: TTFB and caching → hero image priority → render-blocking CSS/JS → third-party scripts → the framework's client bundle. Rewriting the framework before caching the HTML is the classic inversion.

## Performance Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Optimizing for the Lighthouse score | The score is a lab weighting; Google ranks on field data | Watch CrUX at p75 |
| Testing on desktop and a fast connection | Your users and Googlebot's evaluation are mobile | Read the mobile field panel |
| Lazy-loading everything | Lazy hero images directly harm LCP | Lazy-load below the fold only |
| Adding a CDN and calling it done | A CDN fixes distance, not a slow origin render or a 3MB bundle | Diagnose the LCP sub-part first |
| Declaring victory the day of deploy | Field data lags by the 28-day window | Re-check field data four weeks after release |
| Treating CWV as a ranking lever | It is a small tiebreaker between comparable results | Sell it on conversion and bounce, expect ranking only in tight races |
| Deferring the JS that renders content | The content disappears from render and from the index | Defer only what is not needed to paint |
