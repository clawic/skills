# Engines — Choosing and Operating TTS

Engine-neutral by design: dimensions and decision rules, not per-vendor setup — vendor pricing and voice catalogs change too fast to inline; the provider's own docs are the source for current values.

## Decision Dimensions

| Dimension | Trade |
|---|---|
| Local/on-device vs cloud | Local: private, offline, zero marginal cost, flatter prosody. Cloud: best naturalness, per-character cost, and the text leaves the machine — a compliance question, not just a quality one |
| Streaming vs batch synthesis | Streaming starts audio on the first sentence; batch adds the full generation time to perceived latency (SKILL.md Prosody) — interactive use effectively requires streaming or sentence-chunking |
| Quality tier | Providers typically ship standard and neural/HD tiers; the gap is audible on prosody and homographs, barely on short clean sentences — pay for it in long-form and briefings, not in confirmations |
| Language and voice coverage | Multilingual voices switch languages inline (`multilingual.md`); single-language voices anglicize everything they meet |
| SSML tier | full / break-only / none — test protocol in `ssml.md`; record the result in preferences so no session re-tests |
| Cost model | Per-character billing means normalization changes cost, and SSML markup may or may not count as characters — check before templating verbose SSML |

## Voice Selection

- Audition with YOUR content, never the vendor demo: a paragraph containing numbers, a name you actually use, a technical term, and a question. Demo sentences are chosen to flatter.
- Match the voice to the persona register once, then freeze: one voice per persona (SKILL.md rule 6).
- Pin the voice ID and model version where the provider allows — silent provider model updates change how the "same" voice renders (`debug.md`, "sounded different today").

## Voice Cloning

- Never clone a real person's voice without their documented consent — that is impersonation, not personalization; decline and say why.
- A cloned voice copies timbre, not judgment: normalization and prosody rules matter more with a clone, not less, because listeners hold a familiar voice to a higher naturalness bar.

## Operations

- Fallback chain: primary engine → secondary engine → text with a one-line notice (SKILL.md rule 8). Store the chain under the `engine` preference area.
- Cache synthesized audio for fixed utterances (notification templates, standard confirmations): identical string + voice + settings = identical audio; re-synthesis is pure cost and latency.
- Match sample rate to the playback channel — telephony is 8 kHz; feeding it high-rate output produces downsampled mush, and rate mismatch in the player produces chipmunk audio (`debug.md`).
- Per-request character limits vary; chunk at sentence boundaries, never mid-sentence (`debug.md`, "cuts off mid-reply").
- Rate limits under load: degrade to the fallback chain, never to silence.
