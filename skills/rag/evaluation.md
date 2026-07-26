# Evaluation — Knowing Whether It Got Better

Without a golden set, every change is an opinion and every improvement is a story. With one, most changes turn out to be neutral, which is itself the most useful finding available.

**Contents:** [Build the Set First](#build-the-set-first) · [Where Queries Come From](#where-queries-come-from) · [Retrieval Metrics](#retrieval-metrics) · [Generation Metrics](#generation-metrics) · [Calibrating a Score Floor](#calibrating-a-score-floor) · [Paired Comparison](#paired-comparison) · [Sample Size](#sample-size) · [LLM Judges](#llm-judges) · [Regression Gate in CI](#regression-gate-in-ci) · [Online Signals](#online-signals)

**Before any comparison**, read `## Baseline` in `~/Clawic/data/rag/memory.md` and open the golden set its `## Boxes` line names. Comparing against a remembered number is how a regression ships.

## Build the Set First

50 queries, before the first tuning session. It takes an afternoon and it is the difference between engineering and guessing.

Each case: the query verbatim, the `doc_id`s that genuinely answer it, a type label, and where the query came from. Grow toward 200 as production reveals what users actually ask.

Composition that keeps the set honest:

| Type | Share | Why it must be there |
|---|---|---|
| Common questions from logs | ~50% | What the system is for |
| Identifier and exact-term queries | ~15% | The dense-only failure (SKILL.md Rule 5) |
| Multi-part or comparative | ~10% | Routes to decomposition (`agentic.md`) |
| Temporal ("what changed in March") | ~10% | Exposes missing date metadata |
| Negatives — the corpus cannot answer them | ~15% | Without these, refusal cannot be scored and every threshold looks good |

Freeze the set and version it: `support-v3.md`, then `support-v4.md`. Every score names the set version it ran against, or six months of eval history becomes uncomparable.

## Where Queries Come From

- **Production logs** are the best source and the only unbiased one. Sample across the whole distribution, not just the interesting tail.
- **Failures** — thumbs-down, reformulations, abandoned sessions — are the highest-value cases and the ones a hand-written set never contains.
- **Synthetic generation** from chunks is fast and has a specific bias: a question generated from a chunk is answerable by that chunk by construction, so the set measures whether retrieval finds the chunk it was written from, not whether it answers real questions. Use it to bootstrap before there are logs, and dilute it as logs arrive.
- **Domain experts** supply the questions users should be asking and are not yet. Label them by source inside the golden-set file so they score as their own population.

Writing queries after reading the corpus encodes what the system already does well (SKILL.md Traps).

## Retrieval Metrics

Pick the metric that matches the failure being fixed, and report `@k` always.

| Metric | Definition | Reads as | Use for |
|---|---|---|---|
| Recall@k | Fraction of queries whose answer appears in the top-k | Did retrieval lose it? | The gate before any generation work (SKILL.md Rule 1) |
| Precision@k | Fraction of the top-k that is relevant | How much noise reaches the context | Context budget decisions |
| MRR | Mean of `1/rank` of the first relevant result | How high the first good hit lands | Single-answer corpora |
| nDCG@k | Rank-discounted graded relevance | Ordering quality with several relevant documents | Reranking (`reranking.md`) |
| Hit rate@k | Any relevant result in the top-k | Coarse, binary | Quick smoke checks only |
| ANN recall vs exact | Overlap with brute-force top-k | What the index itself costs | Index parameters (`indexing.md`) |

The pair that matters most: recall@`retrieve_k` (did the retriever find it) and recall@`context_k` (did it survive into the prompt). The gap between them is exactly what reranking is buying.

## Generation Metrics

| Metric | Question | How |
|---|---|---|
| Faithfulness / groundedness | Is every claim supported by the context? | Decompose the answer into claims, check each against the context with an NLI model or a judge |
| Answer relevance | Does it address the question asked? | Judge, or embedding similarity between the answer and the question |
| Context precision | Are the chunks that were used ranked highest? | Per-chunk usefulness judgment against the rank order |
| Context recall | Does the context contain everything needed? | Requires a reference answer decomposed into claims |
| Correctness | Is it factually right? | Ground-truth comparison; needs reference answers, which is why the set is expensive to build |
| Refusal accuracy | Refused when it should, answered when it could | The negatives from the golden set |

Faithfulness and correctness diverge, and the divergence is informative: an answer can be perfectly faithful to a context that was wrong, which is a retrieval or corpus problem rather than a generation one.

## Calibrating a Score Floor

Never copy a similarity threshold. Derive it, once per index:

1. Sample 200 random query-document pairs from this corpus — pairs known to be unrelated.
2. Score them with the production embedding model and metric. That is the noise distribution.
3. Take a high percentile of it (95th or 99th) as the floor: below that, a match is indistinguishable from a random pair.
4. Validate against the golden set — the floor should reject the negatives and admit the answerable queries. Adjust, then record the floor and its calibration date in `## Index Registry` in `~/Clawic/data/rag/memory.md` (`memory-template.md`).

Recalibrate whenever the model, the metric, or the corpus composition changes. A threshold with no calibration date is a number nobody dares touch and everybody works around.

Reranker scores need their own floor on their own distribution (`reranking.md`).

## Paired Comparison

- Score both configurations on the **same queries**, and report per-query wins, losses and ties. Two independently sampled sets differ by sampling noise before the change contributes anything.
- A mean that improves while 30% of queries regress is a bad trade, and only the win/loss counts show it (SKILL.md Rule 9).
- Change one variable per run. A row in `evals/<year>.md` carrying two changes is unattributable and should not be written as one row.
- Keep the losers. The list of queries a change broke is the most direct route to understanding what the change actually did.

## Sample Size

Honest bounds, so nobody claims a win they cannot see:

- 50 paired queries surface only large moves — roughly 15 recall points or more.
- A few hundred paired queries is where a 5-point change stops being noise.
- Below 30 queries no comparison is meaningful; the set is a smoke test.
- Segment before believing an aggregate: a 3-point overall gain that is +15 on identifier queries and −2 elsewhere is a hybrid-retrieval result, and the aggregate hides both facts.

## LLM Judges

- Calibrate against human labels on 30-50 cases before trusting a judge on thousands. Report the agreement rate, and re-check it whenever the judge model changes (`llm-as-judge`).
- Pin the judge model and version. A judge that silently upgrades makes every historical score incomparable — the same discipline as the embedding fingerprint (`embeddings.md`).
- Binary or three-point rubrics with explicit criteria beat 1-10 scales, which cluster in the middle and drift between runs.
- Known biases to design against: position (randomize the order of compared answers), length (longer answers score higher), and self-preference (a judge from the same family as the generator).
- Judges are cheap enough to run per change and expensive enough to matter at corpus scale. Budget them like any other per-query cost (`costs.md`).

## Regression Gate in CI

The point of the golden set is that it runs without anyone deciding to run it.

- Run the retrieval metrics on every change to chunking, retrieval, or index configuration. They need no LLM, so they are fast and cheap enough to gate a pull request.
- Fail on a drop beyond the noise band established from repeat runs of the unchanged system — not on any drop at all, or the gate is disabled within a week.
- Run generation metrics on a nightly or pre-release cadence: they need judge calls and cost real money.
- Pin everything the run depends on — set version, model versions, index snapshot — and print them with the result. A score whose inputs are unknown is not a measurement.

## Online Signals

Offline scores predict; production tells the truth.

| Signal | Reads as |
|---|---|
| Reformulation rate | The first answer missed; the strongest implicit negative available |
| Thumbs down with the query attached | A free labeled failure — route it straight into the golden set |
| Empty-result rate | Corpus coverage gap, not a retrieval bug (`retrieval.md`) |
| Refusal rate trend | A jump means the corpus, the index, or the threshold moved (`production.md`) |
| Citation-verification failure rate | Something upstream changed; the earliest available alarm |
| Session abandonment after one answer | Ambiguous between satisfied and given up — never use it alone |

A/B testing a config change needs a metric that moves: task completion or reformulation rate, not a satisfaction survey, and enough traffic that the segment sizes support the claim.

**After every eval run**, append a row to `~/Clawic/data/rag/evals/<year>.md` — date, set name and version, the one variable under test, each metric with its `@k`, p95 latency, cost per query, and the verdict — and when a run becomes the new reference, replace `## Baseline` in `memory.md` with its date. New queries go into a new golden-set version at `~/Clawic/data/rag/goldensets/<name>-v<n>.md`, never into a frozen file, with its `## Boxes` line added in the same turn (`memory-template.md`).
