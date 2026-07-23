---
name: listen
slug: listen
version: 1.0.1
description: Repairs garbled speech-to-text input and learns user-specific mistranscription fixes. Use when voice dictation looks wrong, names or numbers get mangled, or the user corrects transcribed text.
homepage: https://clawic.com/skills/listen
changelog: Complete rewrite with real dictation-repair rules
metadata:
  clawdbot:
    emoji: 👂
    os:
    - linux
    - darwin
    - win32
    displayName: Listen
    configPaths:
    - ~/clawic/listen/
---

Voice input reaches the agent as text that already passed through a speech-to-text engine. Good engines hold 5-10% word error rate on clean conversational English, but the errors concentrate exactly where meaning lives: proper nouns, domain jargon, and numbers. This skill is the repair layer between the raw transcript and your response. Learned corrections persist in `~/clawic/listen/lexicon.md`.

## When To Use

- A message arrived via voice and one token breaks the sentence ("deploy the communities cluster")
- The user repeats, rephrases, or says "no, I said X" after your response or action
- The same name, product, or term gets mangled across sessions and needs a persistent fix
- An STT engine needs vocabulary tuning for a user's recurring domain terms
- Not for transcribing audio files (that is batch transcription work) and not for typed-text typos: keyboard errors are adjacency-based, so phonetic repair misfires on them

## Quick Reference

| Transcript signal | Likely cause | Play |
|---|---|---|
| Common word breaks the sentence's domain | Proper noun replaced by frequent vocabulary | Phonetic match against lexicon, then session context (see Repair Procedure) |
| Number gates an action (amount, count, time) | -teen/-ty confusion (13/30 ... 19/90) | Echo the number as digits in your reply; confirm before irreversible acts |
| User re-sends a nearly identical sentence | Your previous reading was wrong | Diff the two versions; the changed token is the correction; log the pair |
| "period", "comma", "new line" mid-text | Spoken punctuation | Dictating an artifact: treat as command. Conversing: treat as literal word |
| Two words that read as one, or one as two | Segmentation error ("a track" / "attack") | Re-split at syllable boundaries before declaring the token unknown |
| Word fits grammar but not intent ("sine the contract") | Homophone substitution | Same repair path; homophones pass grammar checks, so test against intent, not syntax |
| "um", "uh", "you know" littering the text | Engine transcribed disfluencies | Strip before interpreting; never quote them back to the user |
| 3+ suspect tokens in one message | Mic or noise problem, not a lexical error | Stop piecewise repair; quote your full interpretation back for a yes/no |
| Anything else: transcript reads clean | No error | Respond normally; never mention transcription at all |

## Core Rules

1. **Repair without confirmation when only understanding changes; confirm when the repair changes an action target.** Check: would acting on raw vs repaired text produce different side effects (recipient, amount, file path, send/delete)? Different side effects = confirm first. Same outcome = fix without asking and move on.
2. **Confirm with a candidate, never an open question.** "Did you mean Kubernetes?" costs the user one word; "What did you say?" forces full re-dictation. Offer 1 candidate; 2 only when both fit equally; never 3.
3. **Two-strike promotion.** First observed fix = candidate: apply it but surface it ("...on Kubernetes, got it"). Same fix observed a second time = confirmed: apply from then on without surfacing it. One user rejection at any stage = move the pair to the Never list.
4. **Repair needs two independent signals: phonetic closeness AND context fit.** Phonetic test: strip vowels, collapse doubles, then compare skeletons. "web look" → WBLK vs "webhook" → WBHK is skeleton edit distance 1 (one substitution, L/H); distance ≤2 = neighbor. Context test: the candidate must be a term already in this user's domain (lexicon, recent files, session topic). Either signal alone is a guess, not a repair.
5. **Correction direction is common-word → proper-noun, almost never the reverse.** STT engines are biased toward frequent vocabulary, so a rare word that survived transcription was almost certainly spoken. Never "fix" an odd-looking codename into a dictionary word.
6. **Echo digits for any number that gates an action.** "Booked for 15 (one five) at 8pm." The seven -teen/-ty pairs are the highest-frequency STT number confusion, and a wrong booking count costs more than three extra characters in every reply.

## Repair Procedure

1. Flag the suspect token: it breaks domain, register, or grammar of the surrounding sentence.
2. Generate candidates in priority order: (a) lexicon entries whose wrong-side matches, (b) phonetic neighbors (Rule 4 skeleton test) drawn from session vocabulary, open files, and recent topics, (c) re-segmentation splits/joins.
3. Score by context fit; discard any candidate failing the two-signal test of Rule 4.
4. Route by Rule 1: side effects → confirm with the top candidate; understanding only → substitute without confirmation.
5. After user confirmation or correction, append the pair to the lexicon (next section).

## Correction Lexicon

Store one line per pair in `~/clawic/listen/lexicon.md`:

```
wrong → right | status: candidate|confirmed|never | last seen: YYYY-MM-DD
web look → webhook | confirmed | 2026-07-23
```

- Load the lexicon before interpreting any voice message; apply `confirmed` entries pre-emptively.
- `never` entries are false positives (slang, codenames the user actually says); check them before flagging any token.
- Prune entries not seen for 90 days and keep the active file under ~50 lines: every entry is prompt budget spent on each voice message, and stale pairs from a finished project cause wrong repairs in the next one.

## Engine Tuning

- Persistent mangling of the same 5+ terms means the fix belongs upstream: most engines accept vocabulary biasing (Whisper `initial_prompt`, keyword/phrase boosting in cloud STT). Feed it the proper nouns from the lexicon's right-hand column.
- Pin the language explicitly for multilingual users; auto-detect flips mid-message on code-switching and shreds both languages.
- If transcripts degrade suddenly across all vocabulary, suspect input audio (mic change, background noise), not the engine; ask about the setup once instead of repairing forever.

## Output Gates

Before replying to any voice-sourced message, check:

- Did any repaired token feed an action with side effects? If yes, was it confirmed or lexicon-`confirmed`?
- Is every action-gating number echoed as digits somewhere in my reply?
- Am I mentioning transcription or mishearing anywhere except inside a confirmation question?
- Did this exchange produce a new wrong → right pair, and is it written to the lexicon?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| "Fixing" slang or project codenames | Codenames are deliberately odd; one wrong rewrite teaches the user the agent edits their words | Require both Rule 4 signals; maintain the Never list |
| Asking "what did you say?" | Forces full re-dictation, often while the user is hands-busy, which is why they used voice | Yes/no question with your best candidate |
| Repairing inside dictated artifacts (emails, docs) without marking it | Dictated content is the user's own voice; unmarked edits change what they said | Produce the artifact with repairs applied but uncertain tokens marked for review |
| Keeping corrections only in conversation memory | Lost at session end; the user re-teaches the same name weekly | Persist every pair to `~/clawic/listen/lexicon.md` immediately |
| Applying phonetic repair to typed input | Typing errors follow keyboard adjacency, not sound; phonetic candidates are noise there | Gate this skill on voice-sourced input only |
| Narrating every fix ("your STT said X, I read Y") | Makes the voice channel feel broken and erodes trust in silent repairs | Rule 1 routing: silent when understanding-only, one short confirmation otherwise |
| Promoting a pair to confirmed after one sighting | A single fix may be context-specific; auto-applying it rewrites future valid words | Two-strike promotion (Rule 3); demote on first rejection |

## Related Skills

- **speech-to-text-transcription** - batch transcription of audio and video files with timestamps and speakers
- **voice-notes** - organizing accumulated voice transcripts into a searchable knowledge base
- **talk** - setting up the real-time voice conversation channel this skill repairs
- **audio** - cleaning noisy recordings before they ever reach the STT engine

---
Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/listen.
