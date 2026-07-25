# Search — Text, Fuzzy, and Vector

Three different engines share the word "search" here. Pick by what the query has to do, not by what is already installed.

| Need | Tool | Why |
|---|---|---|
| Exact prefix match, autocomplete on a short field | Anchored regex `/^abc/` on a normal index | Free, no extra infrastructure, index-usable (SKILL.md Query Semantics) |
| Keyword match, no ranking quality requirement | Built-in `text` index | One per collection, works everywhere including self-hosted |
| Relevance, fuzzy, synonyms, facets, multi-field scoring | Atlas Search (Lucene) | The built-in text index cannot do any of these |
| Semantic similarity, RAG retrieval, embeddings | Atlas Vector Search | `$vectorSearch` over an HNSW index |
| Hybrid keyword + semantic | Atlas Search and Vector Search combined | Two searches, fused by rank or score |

## Built-In Text Index — and Its Ceiling

```javascript
db.articles.createIndex({title: "text", body: "text"}, {weights: {title: 3, body: 1}})
db.articles.find({$text: {$search: "\"exact phrase\" -excluded"}}, {score: {$meta: "textScore"}}).sort({score: {$meta: "textScore"}})
```

- One text index per collection, total. Adding a second fails; changing which fields are covered means dropping and rebuilding.
- Stemming is per-language and set at index time; no synonyms, no fuzzy matching, no partial words. A user typing "runing" gets nothing.
- Scoring is a simple term-frequency calculation with the weights above — it cannot express "recent results rank higher" or "boost this field only when the query is short".
- The whole index must be in cache to be fast, and it is large relative to the text it covers.
- Verdict: correct for a keyword filter on a modest corpus; wrong the moment a product manager says the word "relevance".

## Atlas Search

- A Lucene index maintained alongside the collection and fed by change streams: it is eventually consistent, so a document written and immediately searched may not appear for a moment. Never build a read-your-own-writes flow on it (SKILL.md Consistency Model).
- `$search` must be the FIRST stage of the pipeline. Everything after it is normal aggregation, so `$lookup`, `$group`, and projections all compose.
- The index definition is a JSON mapping, not a MongoDB index. `dynamic: true` indexes every field and is right for exploration; a static mapping is smaller, faster, and lets you assign analyzers per field.
- Operators worth knowing: `text` (analyzed match, with `fuzzy: {maxEdits: 1}`), `phrase` (order matters, with slop), `autocomplete` (edge n-grams, needs its own field mapping), `compound` (`must`/`should`/`filter`/`mustNot` — `filter` clauses do not affect score, which is how you keep facet filters from distorting relevance), `range`, `equals`.
- `$searchMeta` returns facet counts and totals without returning documents — the stage that makes a faceted results page one round trip.
- Scoring is tunable: `boost`, `function` (recency decay, popularity), and `constant`. Start with field boosts, add a recency function second, and stop there until you have click data.
- `$search` cannot be combined with a `$match` that should have been a `filter` clause: a `$match` after `$search` filters AFTER scoring and paging, which quietly breaks result counts.

## Atlas Vector Search

```javascript
db.docs.aggregate([
  {$vectorSearch: {
    index: "vector_index",
    path: "embedding",
    queryVector: qv,              // same model, same dimensions, same normalization as at write time
    numCandidates: 200,           // how many the HNSW walk considers
    limit: 10,                    // how many come back
    filter: {tenantId: {$eq: t}}  // only on fields declared as filter type in the index
  }},
  {$project: {text: 1, score: {$meta: "vectorSearchScore"}}}
])
```

- `numCandidates` controls the recall/latency trade: a common starting point is 10-20× `limit`, raised until results stop changing. Too low silently returns worse neighbors — there is no error, only quieter quality.
- Filter fields must be declared as `filter` in the index definition. A filter on an undeclared field is rejected; a filter applied AFTER the search (as a later `$match`) returns fewer than `limit` results and looks like a relevance bug.
- Similarity function (`cosine`, `dotProduct`, `euclidean`) is fixed at index creation and must match how the embedding model was trained. `dotProduct` requires normalized vectors; using it with unnormalized ones ranks by magnitude, not meaning.
- Dimensions are fixed per index. Changing embedding models means a new field, a full re-embed, and a new index — plan it as a migration with dual fields, not an edit (→ `migrations.md`).
- Quantization (scalar or binary) trades a small amount of recall for a large reduction in index memory; on large corpora it is the difference between the index fitting in RAM and not.
- Store the source text alongside the vector in the same document. A vector store that returns ids you then fetch from elsewhere is two round trips and a consistency problem you invented.

## Hybrid Search

- Run `$vectorSearch` and `$search` as separate pipelines over the same collection and fuse the results. Reciprocal rank fusion is the default worth reaching for: score each document as the sum of `1/(k + rank)` across both lists, with `k` around 60, and sort by that sum. It needs no score normalization, which is what makes it robust when the two engines return scores on incomparable scales.
- Use `$unionWith` to run both branches in one aggregation, then `$group` by document id and sum the contributions.
- Judge the change against a fixed query set with known-good answers before and after. Hybrid search always looks better in a demo; only the query set tells you whether it is.

## Choosing and Leaving

- Atlas Search and Vector Search exist only on Atlas. On self-hosted MongoDB the options are the built-in text index, an external engine (→ `elasticsearch` in Related Skills), or a dedicated vector store (→ `vector-databases`).
- The argument for keeping search in MongoDB is one datastore, one consistency story, and filters that can reference any field of the document. The argument for leaving is an ingestion pipeline, analyzers, or relevance tuning that outgrows what the index definition can express.
- Whichever you pick, the deciding evidence is the same: a set of real queries with the answers a human considers correct, scored before and after. Search is the one area where "it feels better" is routinely wrong.
