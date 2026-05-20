---
title: "Building It in Production: System Architecture for High-Precision Legal Retrieval"
date: 2026-05-19
permalink: /posts/2026/system-architecture-legal-retrieval/
tags:
  - retrieval
  - RAG
  - hybrid-search
  - legal-AI
---

The [first post](/posts/2026/why-hybrid-retrieval-fails/) cataloged six ways the standard hybrid retrieval pipeline fails silently on legal corpora. The [second post](/posts/2026/structure-first-retrieval/) defined a four-plane architecture to fix those failures: lexical, dense, structural graph, and symbolic planes, fused with a learned combiner, calibrated, and equipped with principled abstention.

This post is for the developer who has to build it. Not the theory of why the architecture works, but what to actually ship: what services, what storage, what data contracts, what latency budgets, what observability, and how to roll it out without a big-bang release.


## The full system

Two halves. Offline ingest is event-driven: when a source publishes a new document or amendment, the pipeline fans out across parse, segment, extract, normalize, embed, and index. Online query is request-driven: a fixed latency budget, four retrieval planes running in parallel, fusion, rerank, calibrate, respond.

![System architecture diagram showing three zones: offline ingest pipeline (source feeds through layout parser, section segmenter, extractor suite, embedding workers, and rule compiler), shared data stores (lexical index, vector index, graph store, fact table, metadata store, provenance log, rule library), and online query path (query parser, four retrieval planes, score fusion, domain-aware reranker, confidence calibration, evidence bundle).](/images/system-architecture-legal-retrieval.svg)
*The offline ingest pipeline populates seven stores. The online query path fans out across four retrieval planes, fuses and reranks, then returns a calibrated evidence bundle. Dashed lines are metadata filter propagation.*

The system contract at a high level: ingest is asynchronous and idempotent — re-running on the same source produces the same store state. Query is synchronous with a fixed latency budget. All retrieval services are stateless behind their storage. Storage is the only stateful component.

## Ingestion pipeline

The ingest pipeline turns a stream of raw documents into populated indices. Nine logical stages.

**Doc fetcher / watcher.** Polls or subscribes to source feeds. For public statutes, RSS or scraping with content-hash deduplication. For internal manuals, file-system watchers or Git hooks. For regulatory bulletins, often email-to-S3 ingestion with a parser front-end. Each fetched document gets a `doc_id`, a `source_id`, a `fetched_at` timestamp, a content hash, and a raw blob storage path. The fetcher emits a `DocumentFetched` event to a message bus (Kafka, SQS, or Pub/Sub). Idempotency: if the content hash already exists, drop and log. This matters when sources re-publish.

**Layout-aware parser.** For born-digital PDFs, extract text with geometry (font, position, page). For scanned PDFs, run OCR plus layout detection. LayoutLMv3 is a strong general choice; for table-heavy documents, PubTables-1M-trained models are stronger. For HTML, parse the DOM and keep semantic tags. Emit structured blocks with `(text, type, page, bbox, font, parent_id)`.

**Section segmenter.** Reconstruct the document hierarchy from heading patterns, font hierarchy, and TOC structure. Emit a tree of `Section` nodes with `(section_path, level, heading, parent_section_id)`. For statutes this is largely deterministic. For less-structured manuals it is the highest-touch stage.

**Cross-ref extractor.** Scan every section for textual citations. Patterns are domain-specific: `§ 547(c)(9)`, `subsection (b)`, `as defined in section 101(31)`. Each match becomes an edge `(source_clause, target_clause, edge_type, source_span)`. Edge types: `references`, `defines`, `excepts`, `requires`. This is one place where regex craftsmanship pays back many times its cost.

**Defined-term extractor.** Identify definition sections (often signaled by "as used in this section, the term ___ means ___") and produce `(term, definition_clause_id, scope)` triples. Scopes are statute-wide, chapter-wide, or section-wide and must be respected at retrieval time.

**Numeric / date extractor.** Pull monetary amounts, percentages, durations, and dates with the surrounding context window. Each extraction is a row in the fact table: `(clause_id, kind, value, unit, span)`. When an amount is adjusted under a separate provision — like § 104 of the Bankruptcy Code — the extractor produces the adjustment edge and a version-tagged value table.

**Metadata normalizer.** Map document-level metadata onto a canonical schema: jurisdiction, issuing authority, effective_from, effective_to, supersedes, citation_id, version_hash, confidentiality, source_type (statute, regulation, bulletin, manual, opinion).

**Rule compiler (human-in-the-loop).** For the top-queried provisions, a small library of formal rules in a Datalog-like syntax encodes the logic. New statutes do not get formalized automatically. The compiler is a code-review-style workflow: a human (with LLM assistance) writes the rule, links it to source clauses, and a test harness verifies it against labeled cases. Rules are versioned and tagged by effective date.

**Embedding workers.** Compute dense vectors for every clause and section chunk. Embedding refresh is a tracked operation: when the encoder model changes, a re-embedding job runs against the affected corpus partition.

Throughout, the pipeline writes provenance records to the provenance log: which parser version, which extractor version, what source span, what timestamp. This is the audit trail that legal review demands.

**Delta ingestion.** When a statutory amendment arrives, the pipeline re-parses only the affected sections (computed by section-level content hash). Old `Clause` nodes are not deleted. They are marked `effective_to = amendment_date` and the new clause is inserted with `supersedes = old_clause_id`. This preserves as-of-date retrieval. Embeddings and lexical indices are updated for the new clauses; the old ones remain indexed but filtered out for as-of-today queries.

Two engineering disciplines pay back disproportionately. First: idempotency at every stage. Re-running on the same document produces the same store state with no duplicates and no drift. Second: separation of parsing (deterministic) from extraction (probabilistic). Parsing — layout, hierarchy, cross-references — should be deterministic and auditable. Extraction — semantic relations, rules — is allowed to be LLM-assisted but always with a confidence score and human review for high-impact provisions.

## Storage layout

Six stores, each with a clear contract.

**Lexical index.** OpenSearch or Elasticsearch with BM25 scoring, custom analyzers for legal tokenization (preserving `§547(c)(9)` and `` `$7,575` `` as atomic tokens), and learned-sparse fields (SPLADE) if you can afford the index size. Fields per document: `clause_id`, `text`, `jurisdiction`, `effective_from`, `effective_to`, `authority`, `section_path`, `citation_ids`, `version_hash`. Hard filters are index-time fields, not query-time post-filters. This is the difference between fast and slow.

**Vector index.** HNSW is the dominant practical choice for ANN at corpus sizes from 10⁶ to 10⁸. FAISS as the library, Qdrant or Weaviate or Vespa as the managed system, or pgvector for the simplest single-store deployment. Indexes are split by partition (jurisdiction × source_type × effective_year band) so that hard filters can prune partitions before search.

**Graph store.** Two reasonable choices. Neo4j for clean Cypher traversal and graph algorithms. PostgreSQL with recursive CTEs for the structural graph, if the graph is small enough. For 1M documents and the kinds of edges the architecture requires, this is roughly 10⁸ edges, which Postgres can handle on a properly indexed schema. The decision is operational: a separate graph DB adds a service to operate. Both are defensible.

**Fact table.** PostgreSQL. One row per extracted numeric or date: `(fact_id, clause_id, kind, value, unit, version_effective_from, version_effective_to, source_span, parser_version, confidence)`. Indexed by `clause_id` and by `kind, value`. Time-versioning is critical: the same logical fact — say, the § 547(c)(9) threshold — has multiple rows over time.

**Metadata store.** PostgreSQL. One row per document and per clause with the canonical metadata schema. The metadata store is the source of truth for hard filters. The lexical and vector indices replicate metadata fields for query-side filtering. Replication is one-way and event-driven.

**Provenance log.** PostgreSQL for the relational metadata (which document, which span, which parser version, when, what confidence) plus S3-compatible object storage for the raw parsed artifacts. Provenance is append-only. Aim to satisfy W3C PROV semantics: every fact has an agent, an activity, and an entity. Non-negotiable for audit-grade legal systems.

**Rule library.** Versioned files in Git plus a small Postgres index for fast lookup by query type. Each rule is a structured artifact: rule body, premise clauses linked to clause IDs, effective date range, test cases, owner. Rule deployment is a code-review workflow with required test passes.

## The online query path with latency budget

The latency budget for an interactive legal search query should be 1 to 3 seconds end-to-end. Anything faster is unnecessary. Anything slower loses user trust. Here is a typical budget for a single query against a 1M-document corpus:

| Stage | Service | Budget (p95) | Notes |
|-------|---------|--------------|-------|
| Gateway and auth | API gateway | 5 ms | TLS termination, auth, request log |
| Query parsing | Query parser | 50 ms | NER, citation detection, fact extraction, intent routing |
| Filter compilation | Orchestrator | 5 ms | Compile facets to index filters |
| Lexical retrieval | OpenSearch | 80 ms | Top-300 with filters |
| Dense retrieval | Vector index | 100 ms | Top-300 with filters, HNSW ef=200 |
| Graph traversal | Graph service | 50 ms | Seeded typed traversal, depth ≤ 3 |
| Symbolic evaluation | Rule engine | 100 ms | Top-N rule lookups + evaluation |
| Candidate union and dedup | Fusion service | 20 ms | Merge, dedupe by clause_id |
| Score normalization | Fusion service | 10 ms | Min-max per modality |
| Learned fusion | Fusion service | 20 ms | LightGBM scoring on top-1000 |
| Reranker | Reranker service | 250 ms | Cross-encoder on top-100, batched |
| Confidence calibration | Calibrator | 5 ms | Feature compute + isotonic |
| Evidence assembly | Assembler | 30 ms | Pull provenance, format response |
| **Total** | | **~725 ms** | p95; p50 typically ~400 ms |

The query parser is the riskiest stage. If you call a remote LLM for query understanding, you pay 200–500 ms. For an interactive system, run a small local model (a fine-tuned encoder or a distilled instruction model). Alternatively, use deterministic rules plus a smaller LLM only when the rule-based parser is uncertain.

The four retrieval services run in parallel. Their total wall-clock cost is roughly the slowest one, not the sum. With the budgets above, the parallel block takes about 100 ms.

The reranker is the dominant cost. Use a small model (MiniLM-L12 or monoT5-base) and batch aggressively. If precision demands a larger model, route only the top-25 to a heavier cross-encoder. ListT5-style listwise rerankers can be applied to the top-10 for an additional quality bump at modest cost.

The fusion service is cheap because LightGBM scoring on a few thousand candidates is fast on CPU. Keep it on CPU. Do not GPU-allocate this stage.

If you need a hard p95 under 500 ms, the practical levers are: smaller reranker, fewer top-K to rerank, smaller HNSW ef, and aggressive partition pruning via filters.

## Walking a query through the system

Take the query: *"Can a trustee avoid a $6,000 payment made 100 days before filing to an outside vendor of a non-consumer debtor?"*

**T=0 ms.** API gateway accepts the request, authenticates the user, attaches `request_id`, logs the request envelope.

**T=5 ms.** Query parser receives the request. NER tags `outside vendor` (creditor type), `non-consumer debtor` (debtor type). Numeric extractor pulls $6,000 and 100 days. Citation detector seeds § 547. Intent classifier identifies a rule-application question with both retrieval and symbolic relevance.

**T=55 ms.** Orchestrator compiles filters: `jurisdiction = federal`, `as_of_date = today`, `source_type ∈ {statute, opinion, treatise}`. Fans out four retrieval calls in parallel.

**T=55–155 ms.** Lexical, dense, graph, symbolic run concurrently.

- *Lexical:* OpenSearch returns 300 candidates strongly matched on `§ 547`, `90 days`, `transfer`. Returns at ~80 ms.
- *Dense:* HNSW returns 300 candidates including paraphrases and treatise discussions. Returns at ~100 ms.
- *Graph:* Service seeds at § 547(b), walks `excepts` edges to (c)(2) and (c)(9), walks `defines` edges to § 101(31) (insider) and § 101(32) (insolvent). Returns 12 candidates including the exception clauses at ~50 ms.
- *Symbolic:* Rule engine identifies `Avoidable(T)` as the relevant rule, evaluates with the extracted facts. Subsections (b)(4)(A) and (b)(4)(B) both evaluate false; (c)(9) evaluates true. Returns a satisfaction trace at ~100 ms.

**T=155 ms.** Fusion service unions and dedupes candidates, normalizes scores per modality, runs LightGBM with learned weights over `[bm25_norm, dense_norm, graph_support, rule_support, jurisdiction_match, date_valid, authority_weight]`. The top-5 fused candidates: (b)(4)(A), (c)(9), the (b) opening clause, § 101(31) insider definition, and a treatise commentary on the small-transfer exception.

**T=205 ms.** Reranker scores the top-100 with a cross-encoder plus structural features. The top-3 ordering tightens: (b)(4)(A) and (c)(9) at the top, with the symbolic rule trace as a structured annotation.

**T=455 ms.** Confidence calibrator computes features: reranker top score 0.93, margin to second 0.18, modality agreement 4/4, graph support 1, rule support true, contradictions 0. Predicted confidence 0.97.

**T=460 ms.** Evidence assembler pulls provenance for each cited clause, attaches source spans, formats the response.

**T=490 ms.** Response returned: "Not avoidable. Two independent grounds: (1) the 90-day window does not reach 100 days for a non-insider creditor under § 547(b)(4)(A); (2) the transfer is excepted under § 547(c)(9) as a non-consumer transfer below $7,575." Plus the rule trace, citations, jurisdiction, as-of-date, and confidence 0.97.

At every step, structured logs go to the audit pipeline. A reviewer can reconstruct the exact query parse, the candidates from each plane, the fusion scores, the reranker output, the calibrator features, and the final response. This is what "auditable" means in production.

## Service-level concerns

A few things that bite you if you do not plan for them.

**Index refresh during query traffic.** Use blue-green index versions: write to the new index while serving from the old, swap atomically when consistency is verified. Never update an index in-place during query traffic.

**Embedding model upgrades.** When you change the dense encoder, you must re-embed the entire corpus. Run it on a shadow index. Compare retrieval quality on a held-out set before swapping. Plan for at least one re-embedding per quarter.

**Rule library deployment.** Treat the rule library like code: review, test, version, deploy with rollback. A bug in a rule produces a confidently wrong answer with a citation trail. That is the worst possible failure mode. Each rule needs at least one positive and one negative test case in the test harness, with both passing before merge.

**Provenance retention.** Legal review may demand access to the exact retrieval state at the time a question was answered. Retain query logs, fusion features, and responses for the retention period your compliance regime requires. Index by `request_id` and `user_id` for fast lookup.

**Graceful degradation.** If the symbolic engine times out, fall back to retrieval-only with degraded confidence and abstention. If the dense index is unavailable, lexical-only retrieval should still work. If the reranker is down, fused scores without rerank should still produce useful (lower-quality) results. Code the fallbacks explicitly.

**Cost control.** Reranker GPU cost dominates. Use autoscaling on the reranker service with queue depth as the signal. Cache reranker outputs for frequent queries. Legal queries have a long-tail distribution but a meaningful head of frequently-repeated questions, particularly in underwriting and compliance.
