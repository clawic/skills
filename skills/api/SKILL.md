---
name: api
slug: api
version: 1.3.6
description: >-
  REST API reference for 147 services: auth, endpoints, rate limits, pagination, webhooks.
  Use when integrating or debugging a third-party API — Stripe, OpenAI, GitHub, Slack, Twilio,
  and 142 more, with per-service gotchas and curl examples.
homepage: https://clawic.com/skills/api
changelog: Deeper API design heuristics and failure playbooks
metadata:
  clawdbot:
    emoji: 🔌
    requires:
      anyBins:
      - curl
      - jq
    os:
    - linux
    - darwin
    - win32
    displayName: API / Application Programming Interface
---

# API

REST API reference for 147 services: authentication, endpoints, rate limits, and per-service gotchas. User preferences (optional) live in `~/clawic/api/` — the only location this skill writes. If you have data at the old `~/api/` location, move it to `~/clawic/api/`.

## Setup

On first use, read `setup.md` for the lookup procedure and the preferences file format.

## When To Use

- User names a service to integrate ("call the Stripe API", "post to a Slack channel") → jump to its section via API Categories below
- A request that "should work" returns 401/403/429 or silently wrong data → debugging rules and pattern files here
- User must choose between similar providers in a category (email, auth, media) → compare sections in the same category file
- Designing retry, pagination, or webhook handling for any third-party call
- Not for building your OWN API (routing, schema design, versioning) — this skill documents consuming others' APIs

## Architecture

```
apis/                    # API reference files by category
  ├── ai-ml.md           # OpenAI, Anthropic, Cohere, etc.
  ├── payments.md        # Stripe, PayPal, Square, etc.
  ├── communication.md   # Twilio, SendGrid, Slack, etc.
  └── ...

~/clawic/api/            # User preferences (optional)
  └── preferences.md     # Preferred language, frequent APIs, default accounts
```

## Quick Reference

| Situation | Read |
|-----------|------|
| Endpoints/auth for a named service | Its category file — API Categories table below |
| 401/403 that "should work", OAuth flow choice, JWT rejected | `auth.md` |
| Duplicated/missing items across pages, loop never ends | `pagination.md` |
| 429s, timeouts, retries, flaky upstream, circuit breakers | `resilience.md` |
| Receiving events, signature verification, duplicate deliveries | `webhooks.md` |
| Multiple accounts/keys for one service | `credentials.md` |
| Anything else | Core Rules below, then the Official Docs link at the end of each API section |

## API Categories

| Category | File | Services |
|----------|------|----------|
| AI/ML | `apis/ai-ml.md` | anthropic, openai, cohere, groq, mistral, perplexity, huggingface, replicate, stability, elevenlabs, deepgram, assemblyai, together, anyscale |
| Payments | `apis/payments.md` | stripe, paypal, square, plaid, chargebee, paddle, lemonsqueezy, recurly, wise, coinbase, binance, alpaca, polygon |
| Communication | `apis/communication.md` | twilio, sendgrid, mailgun, postmark, resend, mailchimp, slack, discord, telegram, zoom |
| Realtime | `apis/realtime.md` | sendbird, stream-chat, pusher, ably, onesignal, courier, knock, novu |
| CRM | `apis/crm.md` | salesforce, hubspot, pipedrive, attio, close, apollo, outreach, gong |
| Marketing | `apis/marketing.md` | drift, crisp, front, customer-io, braze, iterable, klaviyo |
| Developer | `apis/developer.md` | github, gitlab, bitbucket, vercel, netlify, railway, render, fly, digitalocean, heroku, cloudflare, circleci, pagerduty, launchdarkly, split, statsig |
| Database | `apis/database.md` | supabase, firebase, planetscale, neon, upstash, mongodb, fauna, xata, convex, appwrite |
| Auth | `apis/auth-providers.md` | clerk, auth0, workos, stytch |
| Media | `apis/media.md` | cloudinary, mux, bunny, imgix, uploadthing, uploadcare, transloadit, vimeo, youtube, spotify, unsplash, pexels, giphy, tenor |
| Social | `apis/social.md` | twitter, linkedin, instagram, tiktok, pinterest, reddit, twitch |
| Productivity | `apis/productivity.md` | notion, airtable, google-sheets, google-drive, google-calendar, dropbox, linear, jira, asana, trello, monday, clickup, figma, calendly, cal, loom, typeform |
| Business | `apis/business.md` | shopify, docusign, hellosign, bitly, dub |
| Geo | `apis/geo.md` | openweather, mapbox, google-maps |
| Support | `apis/support.md` | intercom, zendesk, freshdesk, helpscout |
| Analytics | `apis/analytics.md` | mixpanel, amplitude, posthog, segment, sentry, datadog, algolia |

## How to Navigate API Files

Each category file starts with an index table (API name → line number). Read the index, then only the section you need (50-100 lines each):

```bash
head -20 apis/ai-ml.md          # index
sed -n '139,251p' apis/ai-ml.md # one API's section (OpenAI, per the index)
```

## Core Rules

1. **Raw request before client code.** Reproduce with curl first; if curl succeeds and the SDK fails, the bug is in SDK config (base URL, version pin, auth header name), not the API.

2. **Backoff with full jitter, and `Retry-After` overrides it.** `sleep = random(0, min(cap, base × 2^attempt))`, base 1s, cap 30-60s, max 4 retries (AWS "full jitter"). Attempt 3 → sleep is a random value in [0, 8s], not exactly 8s — the randomness is what prevents synchronized retry storms. If the 429/503 carries `Retry-After`, obey it instead.

3. **Retry only what cannot double-execute.** GET/PUT/DELETE are idempotent; retry freely. POST only with an idempotency key — a 500 or timeout on POST may have committed server-side (the charge went through, the response didn't). Details and key lifetime: `resilience.md`.

4. **Status triage in this order: 401 → credential, 403 → permission, 404 → maybe permission too.** 401 = the API doesn't know who you are (missing/expired/malformed token). 403 = it knows you and says no (scope, plan tier, IP allowlist). Some APIs (GitHub among them) return 404 for resources you lack access to, to avoid confirming existence — a "not found" on a resource you know exists is an auth bug, not a URL bug.

5. **HTTP 200 is not success.** Check the body for `error`/`errors` fields (GraphQL always returns 200), and batch endpoints for per-item failures (207 Multi-Status or a 200 with a mixed `results` array).

6. **Both timeouts, always.** Set connect and read timeouts explicitly; a request without them hangs forever on a dead upstream. Values and rationale: `resilience.md`.

7. **Credentials in headers, never in URLs.** Query-param keys land in access logs, proxy caches, and browser history. Env vars only; naming scheme for multi-account setups: `credentials.md`.

8. **Paginate to completion or say you didn't.** Terminate on the API's own signal (`has_more`, next cursor absent) — never on item count alone. If you stop early, state that results are partial.

## Output Gates

Before emitting integration code or a diagnosis, check:

- Every POST/PUT/PATCH example includes `Content-Type: application/json` (or the API's required type)
- No secret appears in a URL, code literal, or logged output — env var references only
- Every pagination loop terminates on the API's `has_more`/cursor signal, not on `len(items)`
- Retry logic distinguishes 4xx (don't retry, fix the request) from 429/5xx (backoff and retry)
- Webhook handler verifies signature on the raw body before parsing (→ `webhooks.md`)

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Missing `Content-Type` on POST | Many APIs parse the body as form-encoded or reject with an unhelpful 400/415 | Always send `Content-Type: application/json` with JSON bodies |
| Trusting the default page size | Defaults are small; you silently process a fraction of the data | Loop until the API's completion signal (→ `pagination.md`) |
| Retrying 400 Bad Request | The request itself is invalid; identical retries burn quota and can trigger abuse detection | Fix the payload; retry only 429/5xx |
| Copy-pasted token fails with 401 | Trailing newline or wrapping quotes from the clipboard corrupt the header | `echo -n`, or trim before use |
| Testing against production keys | Live-mode side effects (real charges, real emails) during development | Use the sandbox key first; prefix conventions in `credentials.md` |
| One giant try/catch around the whole call | 401, 429, and 500 need different responses; a generic catch retries the unretryable | Branch on status class before any retry |

## Scope

This skill is **documentation only**. It provides:
- API endpoint reference
- Authentication patterns
- Code examples for reference

The user manages their own API keys and runs commands themselves.

## External Endpoints

This skill documents external APIs. Example endpoints shown are for the respective service providers (Stripe, OpenAI, etc.).

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/api (install if the user confirms):

- `http` — HTTP request patterns
- `webhook` — Webhook handling
- `json` — JSON processing

## Feedback

- If useful, star it: https://clawic.com/skills/api
- Latest version: https://clawic.com/skills/api

---

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/api.
