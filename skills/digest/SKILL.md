---
name: digest
slug: digest
version: 1.0.1
description: >-
  Curates news, feeds, and industry sources into personalized digest updates. Use for daily
  briefings, news roundups, competitor tracking, or scheduled topic updates. Auto-learns
  format, timing, sources, and depth preferences.
homepage: https://clawic.com/skills/digest
changelog: Better curation rules and summaries
metadata:
  clawdbot:
    emoji: 📰
    displayName: Digest
---

## When To Use

Digest = curate the external world for one specific human. News, industry moves, competitor activity, trends — filtered through their interest profile, delivered in their format at their time. Reach for it when:

- The user wants a recurring **daily or weekly briefing** on topics they care about.
- You need a **news roundup** filtered to one person's interests, not a generic feed.
- The user is **tracking competitors** or an industry and wants ongoing updates on their moves.
- A **scheduled topic update** is due (morning/evening cadence, weekday vs weekend).
- The user asks to **learn and adapt** format, timing, sources, and depth over time.

**Not:** internal business info (→ use Brief), synthesis of documents the user provides (→ use Synthesize)

The test of a real digest: every item carries a why-this-matters-to-you line. A digest that could be sent to anyone is a feed, not a digest.

## Quick Reference

| Situation | Play |
|---|---|
| First digest ever, no preferences | Default format + budget (rule 5), then ask at most two questions: topics + timing |
| Story covered by 3+ independent sources | Promote toward Highlights — convergence is an importance signal (rule 4) |
| Surprising claim from a single source | Ship with explicit hedge ("one source, unverified") or hold one cycle — hold is the default for market-moving or reputational claims |
| Major global news outside user's topics | One line in Worth Noting, never a Highlight — relevance beats magnitude (rule 1) |
| User says "too long" | Cut item count before cutting per-item depth; log the signal |
| Urgent item mid-cycle | Interrupt only on a [confirmed] or [locked] urgent-signal rule; otherwise hold (→ Deliver) |
| No reaction for 3 consecutive digests | Ask one concrete question instead of guessing (→ Learn) |
| Slow news day in their topics | Send the short honest version ("quiet day, 2 items") — never pad |
| Anything else | Run the six-step protocol with current `preferences.md` |

## Core Rules

1. **Relevance beats magnitude.** Rank by interest match first, freshness second, source weight third. A tracked competitor's pricing change outranks a world-news headline outside their topics.
2. **Every item is attributed.** No source you can name = item does not ship. Never blend your own inference into a summary unmarked — prefix it: "Likely:" or "My read:".
3. **Exclusions are binary, interests are weighted.** One explicit "don't care about Y" beats any number of inferred interest signals for Y. An excluded topic appears only when it intersects a confirmed interest, flagged as such ("normally excluded; included because it hits X").
4. **Dedupe before ranking.** Same story from N outlets = one item; N itself is rank input — 3+ independent sources covering one story promotes it toward Highlights.
5. **Fixed budget forces curation:** default 5-8 items, hard max 3 Highlights. Adding a 9th item means deleting one. Worked example: 12 candidates → 3 Highlights (top interest-match), 5 body items, remaining 4 dropped or compressed into one Worth Noting line.
6. **Preference changes climb a ladder** — pattern → confirmed → locked, thresholds in `preferences.md`. Never jump from a single signal to locked.

## Protocol

```
Source → Filter → Prioritize → Format → Deliver → Learn
```

### 1. Source

Pull from configured feeds, news, social, and industry sources per `preferences.md`. Default source weighting when unset: primary (announcements, filings, papers) > original reporting > aggregators > social commentary. Social is a discovery layer, not a citation layer — trace anything found there back to a primary or reporting source before it ships.

### 2. Filter

- Topic match includes; exclusion match drops (rule 3).
- Recency windows per content type: canonical values in `dimensions.md` → Sources.
- Credibility: single-source surprising claim → hedge or hold (Quick Reference row 3).

### 3. Prioritize

Rank by the user's weighting profile (`dimensions.md` → Prioritization):

- Interest match, then freshness, then source weight (rule 1).
- Breaking-first vs calm-curated ordering is a learned preference, not a default. Until learned, lead with the highest interest-match item.
- Bury, don't delete: borderline items compress to one line in Worth Noting.

### 4. Format

Apply learned format (`dimensions.md` → Format): channel, medium, structure, length, tone, visuals.

- Per-item shape: headline → 1-line summary → why it matters to this user → (source).
- Length pressure cuts item count before per-item depth: 8 items with a why-line beat 15 bare headlines the user won't expand.

### 5. Deliver

- Timing per preference: morning/evening/both, weekday vs weekend, user's timezone.
- Mid-cycle interrupt only for items matching a [confirmed] or [locked] urgent-signal rule. No urgent rule learned yet → never interrupt; propose the rule in the next digest instead.
- Slow day: short honest digest. Padding trains the user to skim, which destroys the Highlights section's authority.

### 6. Learn

After each delivery, capture signals into `preferences.md` (thresholds there):

| Signal | Adjustment |
|---|---|
| "Too long" / stops reacting after item 4 | Cut item count next digest; log length signal |
| "Missed X" | Add X's sources and keywords; diagnose why it was filtered out |
| "Don't care about Y" | Exclusion, effective immediately (rule 3) |
| "Love this" / forwards an item | Reinforce that topic and format; note what was different about it |
| Asks follow-ups on a topic | Depth signal: raise per-topic depth |
| No reaction for 3 consecutive digests | Ask one concrete question (e.g., "keep the morning slot?"); don't guess |

## Preference System

`preferences.md` = current learned state (empty sections mean defaults apply). `dimensions.md` = every trackable dimension with its default. Read preferences before every digest; write new signals after every delivery.

## Output Format (Default)

```
📰 [DIGEST TYPE] — [DATE/TIME]

🔥 HIGHLIGHTS
• [Item — 1-line summary + why it matters to you (source)]
• [Second item]

📋 FULL DIGEST
[Items in per-item shape, ordered per weighting profile]

💡 WORTH NOTING
[Borderline items, one line each]

---
Sources: [count] | Next digest: [time]
```

Adapt entirely to learned preferences once they exist; the template above is only the cold-start default.

## Output Gates

Before sending, verify:

- Every item has a source and a why-it-matters line?
- ≤3 Highlights, each strong enough that the user could read only those?
- Zero exclusion-list items, unless flagged per rule 3?
- Channel and time match current `preferences.md`?
- Signals from the previous delivery recorded in `preferences.md`?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Leading with globally important news | User already saw it everywhere; the digest adds zero | Lead with niche relevance; big news gets one line (rule 1) |
| Padding slow days | Trains skimming; erodes trust in Highlights | Short honest digest ("quiet day") |
| Locking a preference from one comment | One signal may be mood, not preference | Climb the ladder in `preferences.md` (rule 6) |
| Reading silence as satisfaction | Ignored digests look identical to loved ones | No reaction for 3 consecutive digests → ask one question |
| Unmarked opinion inside summaries | User can't separate reporting from your read | Attribute claims; prefix inference (rule 2) |
| Resurfacing yesterday's story unchanged | Repetition reads as a broken filter | Only resurface as "Update:" + what changed |
| Interrupting off-schedule on your own judgment | "Urgent" without a learned rule is noise at a bad hour | Interrupt only per learned urgent-signal rule (→ Deliver) |

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):

- **Brief** (`brief`) — Jump here when the user wants internal business information summarized rather than external news curated.
- **Synthesize** (`synthesize`) — Jump here when the user provides documents to be distilled, instead of asking you to source and filter the outside world.

---

*References: `dimensions.md`, `preferences.md`*

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/digest.
