# Security — Access, Isolation, Injection, Erasure

A RAG system is a machine for showing people documents. Every security question in the domain reduces to: which documents, to whom, and can anyone prove it afterwards.

**Contents:** [Access Control at Retrieval](#access-control-at-retrieval) · [Permission Sync](#permission-sync) · [Multi-Tenant Isolation](#multi-tenant-isolation) · [Prompt Injection From Indexed Documents](#prompt-injection-from-indexed-documents) · [PII](#pii) · [The Right to Erasure](#the-right-to-erasure) · [Data Residency and Third Parties](#data-residency-and-third-parties) · [Audit Trail](#audit-trail) · [Compliance Regimes](#compliance-regimes) · [Red Flags](#red-flags) · [Audit Checklist](#audit-checklist)

**Before designing any permission model**, read `## Corpus` in `~/Clawic/data/rag/memory.md`: the `Access field` column names which metadata key carries permissions for each source, and a source with no access field in a permissioned system is a finding.

## Access Control at Retrieval

Filter **during** the search, never after, and never in the prompt.

- **Never post-filter for permissions.** Retrieving 30 chunks and dropping the ones the user cannot see means the user gets 4 results and the model was, for a moment, handed 30 (SKILL.md Rule 6). It also leaks through timing and result counts.
- **Never delegate to the model.** "Only answer using documents the user is allowed to see" is a request, not a control.
- **Derive the filter from the session**, server-side, on every query. Not from the conversation, not from the rewriter's output, not from anything the user can type (`conversation.md`).
- **Fail closed.** A missing or unresolvable permission context returns nothing, never everything. The default value of an unset filter variable is the most dangerous line in a RAG codebase.
- **Chunks inherit document permissions at ingestion**, and the field is mandatory. A chunk indexed before the field existed is invisible to the filter and therefore either always hidden or always visible — check which, because both are bugs (`indexing.md`).

Two shapes work: a group or clearance field checked against the session's groups, or a per-document ACL list intersected with the session identity. High-cardinality ACL lists push the query toward brute force, which is a design constraint, not an implementation detail.

## Permission Sync

The corpus is a copy, so its permissions are a copy, and copies go stale.

- Re-sync permissions from the source system on a cadence, not only at ingestion. Someone leaves a team on Monday and the index still says they can read it in December.
- A revocation must propagate faster than a document update. Treat permission changes as their own event stream where the source system offers one.
- Deleted upstream documents keep their content in the index until a reconcile pass notices they are gone (`production.md`).
- Caches hold retrieved content past a revocation: include the permission context in every cache key, or do not cache retrieved content at all.
- Put the resync in the `## Due` table so it is a scheduled thing rather than a good intention.

## Multi-Tenant Isolation

| Approach | Isolation strength | When |
|---|---|---|
| Index or collection per tenant | Strongest: a leak requires querying the wrong collection | Few large tenants, regulated data |
| Namespace or partition per tenant | Strong where the store implements a hard partition | The default when supported |
| Metadata filter on a shared index | Only as strong as the filter code and its tests | Many small tenants, with an automated isolation test |

The test that makes the third option defensible, and that the first two also deserve: seed tenant B with a canary document containing a unique string, query as tenant A for that string on every build, and fail the build on a single hit. Run it against the reranker and the cache too — a shared cache reintroduces cross-tenant leakage that the store prevented.

Isolation applies to more than retrieval: per-tenant rate limits, per-tenant logs, and per-tenant eval sets. A shared semantic cache is a cross-tenant channel by construction (`production.md`).

## Prompt Injection From Indexed Documents

The corpus is user-supplied content. Anyone who can add a document — a support ticket, a wiki page, an uploaded PDF — can attempt to instruct the model.

Defenses, in order of effectiveness:

1. **Structural separation.** Retrieved content sits in a fenced, labeled block, and the system prompt states that content inside it is data describing the world, never instructions (`generation.md`).
2. **Least privilege on the generator.** An assistant that only writes text can only be made to write bad text. Injection becomes serious the moment retrieval feeds an agent with tools — which is exactly the agentic path (`agentic.md`).
3. **Output constraints.** Validate the answer's shape: cited ids must exist in the context, links must resolve to indexed sources, no tool call the route did not authorize.
4. **Ingestion-time scanning.** Flag documents containing instruction-shaped text ("ignore previous", "system:", role markers, invisible Unicode, zero-width characters, white-on-white text in PDFs). Quarantine rather than silently strip, so someone reviews it.
5. **Provenance in the answer.** Showing which document a claim came from turns a successful injection into something a human can spot.

What does not work: a blocklist of injection phrases. Paraphrase defeats it, and the false positives land on legitimate documents about prompt injection.

**Exfiltration is the other half.** An injected instruction can try to make the model emit data from another retrieved chunk into a URL, an image reference, or a tool call. Never render model-supplied URLs or images without validating them against the indexed sources.

## PII

Decide per corpus, at ingestion, and record the decision in `~/Clawic/data/rag/artifacts/decision-access-control.md`.

| Option | Trade |
|---|---|
| Index as-is with access control | Simplest; correct when the corpus is internal and permissions are real |
| Redact before embedding | Placeholders survive retrieval, so the answer is safe and the search for "Ana's contract" no longer works |
| Separate index for sensitive content | Clean boundary, two pipelines to maintain |
| Pseudonymize with a reversible map | Keeps searchability; the map is now the most sensitive asset in the system |

Detection combines pattern matching for structured identifiers (card numbers, national ids, emails, phones) with an NER model for names and addresses. Neither is complete. Measure recall of the detector on a labeled sample before claiming the corpus is clean, and record the number rather than the claim in `~/Clawic/data/rag/artifacts/decision-access-control.md`.

Consequences that people miss: PII propagates into **logs, eval sets, caches and golden sets**. A golden set built from production queries is a personal-data store. Redact there too (`evaluation.md`).

## The Right to Erasure

Erasure is not "delete from the index". Enumerate every copy:

| Copy | How it is erased |
|---|---|
| Source document | Upstream, by the owning system |
| Chunks in the vector index | Delete by `doc_id` filter (SKILL.md Rule 8) |
| Chunk text in a payload store or system of record | Same delete, different store — easy to forget |
| Caches: retrieval, semantic, answer | Invalidate; keys must make this possible |
| Query and answer logs | Retention policy, or targeted deletion |
| Golden sets and eval fixtures | Manual, and the one that is always missed |
| Backups and index snapshots | Documented retention window, disclosed |
| The embedding vector itself | Deleted with the chunk. Vectors are derived personal data, not anonymized data |

Test the procedure end to end on a real document and record the elapsed time. A documented erasure path nobody has run is not a path.

Tombstones matter here: a store that marks rather than reclaims still holds the bytes until compaction (`indexing.md`).

## Data Residency and Third Parties

- Every hosted embedding, rerank and generation call sends content out of the perimeter. Enumerate the providers and what each one receives — chunks, queries, or both. A reranker sees full passages, which surprises people who only audited the generator.
- Check retention and training-use terms per provider, and pin the region where the provider offers one. The vector store's region is not the embedding provider's region.
- Self-hosting the embedding model keeps documents inside and still leaves the generator outside unless that is self-hosted too. Partial perimeters are a legitimate design; an undocumented one is not.
- Under a regime that requires it, a data processing agreement or BAA is required with each processor, including the reranker.

## Audit Trail

Log per query, immutably: timestamp, principal, tenant, the query as received, the permission filter applied, retrieved `chunk_id`s with scores, the ids that reached the context, the answer, cited ids, and the model versions used.

- The `chunk_id` list is what makes "who saw this document" answerable. Without it, an access incident cannot be scoped and must be assumed total.
- Append-only storage with a documented retention period matched to the regime.
- Logs contain the query text and therefore contain personal data. They are inside the erasure scope and inside the access-control scope — audit logs need their own permissions.

## Compliance Regimes

Set `compliance_regime` and these become requirements rather than suggestions.

| Regime | What it forces |
|---|---|
| GDPR | Lawful basis for processing; erasure across every copy above; residency; accuracy, so stale personal data in an index is itself a violation; processor agreements with every provider |
| HIPAA | BAA with every processor including embedding and rerank providers; encryption at rest and in transit; access logging per record; retention matched to the regime |
| SOC 2 | Documented access control with evidence; encryption; immutable audit trail; change management on the index and prompts; incident response |
| None | The controls above are still the defaults; the regime changes what is auditable, not what is correct |

## Red Flags

Anything in this table suspends the design work and gets raised before anything ships.

| Signal (observable) | Suspicion | Action |
|---|---|---|
| Permission filter derived from anything the user can type | Privilege escalation by prompt | Move derivation server-side, from the session, today |
| Chunks in the index lacking the access field | Documents visible to everyone or no one | Count them, backfill or reindex before launch |
| A user reports seeing content they should not | Active leak | Freeze retrieval for that surface, scope with the `chunk_id` logs, then fix |
| A shared cache keyed without tenant or permission context | Cross-tenant leakage past a correct filter | Add the key components or disable the cache |
| Indexed document containing role markers or invisible Unicode | Injection attempt | Quarantine the document, review the ingestion path |
| No `doc_id`-based delete path | Erasure impossible | Blocks any personal-data corpus, regardless of regime |
| Chunk text or credentials found under `~/Clawic/data/` | Confidential content written where it must never be | Remove it, replace with pointers, and check what else that session wrote |

## Audit Checklist

| Check | Passing looks like |
|---|---|
| Permission filter applied inside the search, from the session | Verified in code and by an isolation test in CI |
| Access field present on 100% of chunks | A count query returning zero missing |
| Tenant isolation test | Canary query fails the build on any cross-tenant hit |
| Cache keys include tenant and permission context | Inspected, per cache |
| Retrieved content fenced and labeled as data | Present in the system prompt, verified in the rendered prompt |
| Generator has no tools it does not need | Tool list reviewed against the surface |
| Erasure path tested end to end | A dated run with the elapsed time recorded |
| Provider inventory with region and retention terms | One row per processor, including the reranker |
| Audit log with retrieved chunk ids | Immutable, retention set, its own access control |
| Nothing secret or confidential written to local data files | Pointers only (`memory-template.md`) |

**After a security decision or an audit pass**, record it in `~/Clawic/data/rag/artifacts/decision-access-control.md` — the permission model, the isolation approach, the PII handling option chosen per corpus with the measured detector recall, the erasure procedure with its tested elapsed time, and the provider inventory — with its `## Boxes` line in the same turn. Put the permission-resync cadence and the isolation-test cadence in `## Due`, and any gap found into `## Known Failures` with `Still open: yes` until it is closed (`memory-template.md`). Never write chunk text, personal data, or credentials into any of these files.
