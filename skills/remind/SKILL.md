---
name: remind
slug: remind
version: 1.0.1
changelog: Smarter timing and phrasing guidance
description: Reminds the user of commitments they already know about — meetings, deadlines, promises — at learned lead times and in their preferred style. Use when the user says remind me, mentions a future commitment, or gives feedback on reminder timing.
homepage: https://clawic.com/skills/remind
metadata:
  clawdbot:
    emoji: ⏰
    displayName: Remind
---

## When To Use

**Remind = "Just a reminder that..."** — surfacing something the human already knows, at the moment it becomes actionable.

- Planned events, deadlines, commitments they set themselves
- Promises they made to other people
- Things they told you they were afraid to forget

**The litmus test:** could they have written it in their own calendar? Then it's a reminder. Did the world just change (news, outage, price drop)? That's Alert. Generic status? That's Notify. If "just a reminder that..." reads unnaturally in front of the message, don't send it as one.

## Core Rules

Learn what to remind, when, and how — through observation, never silently.

**Core Loop:**
1. **Detect** — notice remindable commitments (`triggers.md` defines what qualifies)
2. **Evaluate** — check stored preferences below; fall back to `timing.md` defaults
3. **Remind** — deliver while they can still act, not merely know (→ Reminder Components)
4. **Observe** — log reactions: "too early", "I knew", "thanks, I forgot", silence
5. **Confirm** — after 2 consistent signals, propose the adjustment in one line
6. **Store** — write the preference below only after they accept, or on explicit instruction

**Signal ladder** (canonical — other files point here):
- Explicit instruction ("always ping me the day before") → store as `(confirmed)` immediately
- 2 consistent observed reactions → store as `(pattern)`, propose the change
- Accepted proposal, or a 3rd consistent reaction → upgrade to `(confirmed)`
- 1 contradicting reaction on a `(confirmed)` entry → downgrade to `(pattern)`, re-observe

## Quick Reference

| Situation | Play |
|---|---|
| "Remind me to X at/in Y" | Create exactly as stated; explicit override beats every learned preference and default |
| "Don't let me forget X" | Create with an extra earlier stage — the phrase signals stakes, not timing |
| Commitment mentioned, no ask ("I'll send it Friday") | First 2 in a category: offer ("Want a nudge Friday morning?"); after 2 acceptances, create silently |
| Reaction to a past reminder | Feed the signal ladder; adjust via the lead ladder in `timing.md` |
| They acknowledged the reminder | Done — no repeat unless the entry is under **Always** |
| **Default: unsure if remindable** | High stakes → create and say so; low stakes → skip and watch what happens |

## Reminder Components

- **What** — the commitment, phrased as the *action*, not the event: "Leave by 2:15 for the 5pm flight", never "Flight at 5pm". If the reminder arrives when the action is no longer possible, the timing was wrong regardless of the event time.
- **When** — lead time before the action must start (`timing.md`)
- **How** — tone and length from **Style** below; default: one line, what + when + the single next step

## Entry Format

`category: preference (level) [notes]`

Levels: `(pattern)` and `(confirmed)` per the signal ladder above.

Examples:
- `meetings: 30 min before (confirmed)` — learned override of the 15-min default
- `deadlines: 1 day + morning-of (pattern)`
- `family events: 1 week before (confirmed)`
- `daily standups: skip (confirmed) [muscle memory]`

---

### Timing
<!-- Lead times by category -->

### Style
<!-- Brief vs detailed, casual vs formal -->

### Always
<!-- Remind even after acknowledgment or apparent awareness -->

### Skip
<!-- Never remind; correct Skips are wins, not gaps -->

---

## When NOT to Remind

- They mentioned it in the current conversation — aware right now. New session → fair game again.
- It's new information → Alert. Generic status → Notify.
- They already acknowledged this reminder — a repeat is nagging, unless listed under **Always**.
- Routine they never miss — reminding transfers ownership of the habit to you; the day you're offline, the habit fails.
- Low stakes + uncertain → skip and learn from what happens. A reminder system that works sends *fewer* reminders each month, not more.

## Output Gates

Run before every reminder goes out:
- [ ] Do they already know this? (No → it's Alert or Notify, not Remind)
- [ ] Can they still act on it after this arrives? (No → too late; log it as a missed lead for the ladder)
- [ ] Has this exact reminder been acknowledged? (Yes → drop unless in **Always**)
- [ ] Does a **Skip** entry or quiet hours cover it? (`timing.md`)
- [ ] Is it phrased as the action, not the event?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Reminding about everything detected | Fatigue: they tune out, and the one critical reminder dies with the noise | Skip low stakes by default; let reactions promote categories |
| Reminding at event time | Awareness without agency — nothing left to do but feel bad | Lead counts back from when the *action* starts (`timing.md` formula) |
| Repeating after an acknowledgment | Reads as nagging; erodes the trust the whole skill runs on | One ack = done; **Always** is the only exception |
| Treating "thanks" as "send more" | Politeness is not preference | Only timing words ("early", "late", "perfect") and actual misses move the ladder |
| Storing preferences silently | Wrong guesses compound invisibly; they can't correct what they can't see | Propose in one line before writing anything `(confirmed)` |
| Stretching lead time after one miss | One miss may be a bad day; overcorrection breeds "too early" complaints | Move one ladder step per 2 signals, never multiply |

---

*Empty sections = still learning. Observe and fill.*

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/remind (install if the user confirms):
- `alerts` - New or urgent events they don't know about yet
- `notify` - Delivery mechanics: channels, batching, fatigue control
- `schedule` - Executing recurring tasks, not recalling them
- `memory` - Long-term storage beyond reminder preferences

## Feedback

- If useful, star it: https://clawic.com/skills/remind
- Latest version: https://clawic.com/skills/remind

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/remind.
