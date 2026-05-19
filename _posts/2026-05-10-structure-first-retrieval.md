---
title: "Structure-First Retrieval: The Scientific Case"
date: 2026-05-10
permalink: /posts/2026/structure-first-retrieval/
tags:
  - retrieval
  - RAG
  - hybrid-search
  - legal-AI
---

The [previous post](/posts/2026/why-hybrid-retrieval-fails/) cataloged six ways the standard hybrid retrieval pipeline fails silently on legal corpora. All six come down to the same root cause: the pipeline operates on flattened text, but legal queries are about structured facts. This post describes the architecture I built to fix that.

## Recap

The standard hybrid pipeline (BM25 for lexical match, a bi-encoder for dense retrieval, a cross-encoder for reranking) produces wrong-but-confident answers on legal corpora. Six failure modes, all illustrated on 11 U.S.C. § 547 (the Bankruptcy Code's preference action), recur on essentially every corpus with version, jurisdiction, hierarchy, exception, and threshold structure:

1. **Version drift.** Multiple historical versions of a statute coexist in the corpus; no component of the pipeline conditions on effective date.
2. **The exception you never see.** Chunk boundaries break cross-references, so the rule is retrieved without its qualifying exception.
3. **Paraphrase loss of load-bearing words.** Embeddings treat "shall" and "may" as near-synonyms; in law they are opposites.
4. **Authority bleed-through.** No model of the hierarchy of authorities (federal vs state, statute vs regulation, binding vs persuasive).
5. **Temporal and numeric reasoning.** The retriever surfaces rules but cannot evaluate them against the facts in the query.
6. **Confidence is not correctness.** Reranker scores measure topicality, not legal correctness, and are systematically over-confident on the failure modes above.

The common pattern: legal queries are about *structured facts* (dates, jurisdictions, thresholds, hierarchies, logical predicates) and the standard pipeline operates entirely on flattened text. The fix is not a better text-text scorer. The fix is to preserve structure on ingest and use that structure at retrieval time.

Structure-first retrieval, the design this post describes, means: a retrieval architecture in which (a) the corpus is decomposed on ingest into typed, versioned, citation-linked artifacts; (b) the retriever has four planes of candidate generation (lexical, dense, structural-graph, symbolic) rather than one or two; (c) candidate fusion is a learned function over modality-specific scores plus structural features; (d) reranking uses domain features in addition to neural scores; and (e) calibration and abstention are first-class components rather than afterthoughts.

## Architectural overview

![Structure-first retrieval: four-plane query architecture showing Query Parser feeding into P1 Lexical, P2 Dense, P3 Graph, and P4 Symbolic planes, then Score Fusion, Domain-aware Reranker, Calibrated Confidence, and Evidence Bundle.](/images/structure-first-architecture.svg)
*The four-plane query pipeline. Each plane contributes orthogonal signal; fusion exploits that orthogonality.*

The diagram shows the online query path only. Offline, ingest produces five indexes: a lexical inverted index, a dense vector index, a normalized metadata store, a citation graph, and a fact table of extracted numerics and dates. At query time, all four planes are filtered by the metadata store (jurisdiction, effective date, authority weight applied as hard filters before any plane returns candidates). Everything from ingest is parsed, not interpreted: every fact, edge, and field has a literal source span. No LLM hallucination, no embedding drift.

## Plane 1: Lexical retrieval

The lexical plane is BM25 (and optionally a learned sparse retriever such as SPLADE). It is the most important plane for legal text and the most under-appreciated.

BM25 scores a query $q = (q_1, \ldots, q_m)$ against a document $d$ as

$$
\text{BM25}(q, d) = \sum_{i=1}^{m} \text{IDF}(q_i) \cdot \frac{f(q_i, d) \cdot (k_1 + 1)}{f(q_i, d) + k_1 \cdot (1 - b + b \cdot \frac{|d|}{\text{avgdl}})}
$$

where $f(q_i, d)$ is the term frequency of $q_i$ in $d$, $\|d\|$ is the document length, $\text{avgdl}$ is the average document length, $k_1 \in [1.2, 2.0]$ controls term-frequency saturation, and $b \in [0, 1]$ controls length normalization. The IDF term is

$$
\text{IDF}(q_i) = \log \frac{N - n(q_i) + 0.5}{n(q_i) + 0.5}
$$

with $N$ the corpus size and $n(q_i)$ the document frequency of $q_i$.

Two observations matter for legal corpora. First, the IDF term rewards rare tokens disproportionately, which is exactly the right inductive bias for legal text: statutory citation tokens ("547(c)(9)"), monetary amounts ("$7,575"), and proper nouns of statutes and agencies are precisely the tokens that should dominate scoring. Second, the saturation function $\frac{f \cdot (k_1 + 1)}{f + k_1 \cdot (\cdot)}$ damps the contribution of common-word repetition, which keeps boilerplate from dominating.

Two non-default choices matter for legal corpora. First, tokenize so that statutory citations and dollar amounts are preserved as single tokens. The string `§547(c)(9)` should not be split, and `$7,575` should not be split. Token preservation alone is responsible for substantial precision gains because it makes IDF rare in the right places. Second, optionally layer SPLADE on top:

$$
\text{SPLADE}(q, d) = \sum_{j \in V} \max\left(0, \log\left(1 + w_j^{(q)}\right)\right) \cdot \max\left(0, \log\left(1 + w_j^{(d)}\right)\right)
$$

where $w_j^{(q)}$ and $w_j^{(d)}$ are learned expansion weights over the vocabulary $V$. SPLADE adds semantic expansion (so "vehicle" can match documents that say "automobile") while remaining sparse and interpretable. The cost is index size; the benefit is closing some of the vocabulary mismatch gap without paying the full cost of dense retrieval.

The lexical plane fixes failure mode 3 (load-bearing words): if the query says "shall," BM25 with proper tokenization will rank passages containing "shall" above paraphrases containing "will" or "must." Failure mode 1 (version drift) is not fixed by the lexical plane (versioned text matches identically) and is addressed instead with metadata filters threaded through every plane.

## Plane 2: Dense semantic retrieval

The dense plane converts the query and each passage into a vector via an encoder $E$ and retrieves nearest neighbors:

$$
\text{score}_{\text{dense}}(q, d) = \cos(E(q), E(d)) = \frac{E(q) \cdot E(d)}{\|E(q)\| \cdot \|E(d)\|}
$$

Two design choices matter. The first is bi-encoder vs late-interaction. A bi-encoder produces a single vector per passage; retrieval is fast and cheap but loses fine-grained token-level interaction. A late-interaction model (ColBERT and its descendants) keeps a vector per token and scores via

$$
\text{score}_{\text{ColBERT}}(q, d) = \sum_{i=1}^{|q|} \max_{j \in \{1, \ldots, |d|\}} E_q(q_i) \cdot E_d(d_j)
$$

This is the MaxSim operator. ColBERT-style retrieval costs more in index size and online compute but captures subtle token-level matches that bi-encoders miss. On legal text (where the difference between "any" and "such" can matter) late interaction has measurable precision benefits.

The second choice is the encoder itself. BGE-M3 is a strong default: it produces dense, sparse, and multi-vector representations from one model, supports long context, and is competitive on BEIR. For higher precision at higher cost, a domain-adapted late-interaction model trained on legal pairs is the better choice. The training recipe (GPL-style pseudo-labeling, Promptagator synthetic query generation, hard-negative mining within the same statute or same jurisdiction) is well-supported in the literature and works with the small label budgets typical in enterprise legal projects.

The dense plane partly addresses failure mode 3 in the opposite direction from BM25: it tolerates paraphrase. The risk is that this tolerance over-rides exact matching on load-bearing words. The remedy is fusion (Plane 5), not picking one plane.

The dense plane does not, on its own, fix any of the six failure modes. It is necessary because vocabulary mismatch is severe in legal text (a citation can be referred to as "section 547(c)(9)," "§ 547(c)(9)," "the small-transfer exception," or "the de minimis safe harbor"). But it must be paired with structural and symbolic planes to be safe.

## Plane 3: Structural graph from parsed text

This plane is where structure-first retrieval differs most from hybrid retrieval. The structural plane is a graph $G = (V, E)$ whose nodes are typed and whose edges are parsed deterministically from the document itself, not extracted by an LLM. Formally:

- $V$ contains node types: `Document`, `Section`, `Clause`, `DefinedTerm`, `Citation`, `Threshold`, `Date`, `Jurisdiction`, `Authority`.
- $E$ contains edge types:
  - `contains(Section_i, Section_j)` for hierarchy (from headings)
  - `cross_refs(Clause_i, Clause_j)` for textual cross-references (from "subsection (c)" or "§ 101(31)")
  - `defines(Clause_i, DefinedTerm)` from definition markers
  - `uses(Clause_i, DefinedTerm)` from term occurrences
  - `supersedes(Clause_i, Clause_j)` from amendment metadata
  - `effective(Clause_i, [t_from, t_to])` from effective-date metadata
  - `applies_in(Clause_i, Jurisdiction)` from publishing authority

Every node and edge carries `(source_id, source_span, parser_version, confidence, prov_id)`. The crucial property: extraction is deterministic and auditable. A regex or a structure-aware parser produces these edges. No LLM hallucination, no embedding drift.

Graph retrieval, given seed nodes from the query, is a typed traversal:

$$
\text{Retrieve}_G(S, \tau, t) = \{\, v \in V \,:\, \exists \text{ path } p \text{ from } s \in S \text{ to } v, \text{type}(p) \in \tau, \text{valid}(v, t) \,\}
$$

where $S$ is the seed set (from query parsing; "§ 547(c)(9)" produces a seed), $\tau$ is the allowed set of edge types for this query (citation walks, definition walks, exception walks), and $t$ is the as-of-date used to filter via the `effective` edges.

The structural plane fixes failure mode 2 (the exception you never see). When the query parser sees "§ 547," the seed set is the (b) node. The traversal follows `cross_refs` edges to (c)(9), because (b) opens with "Except as provided in subsections (c) and (i)." This is a deterministic textual cross-reference, not a learned association.

It also partially fixes failure mode 1 (version drift): the `effective` edge filter restricts retrieval to clauses valid at the query's as-of-date. And it partially fixes failure mode 4 (authority bleed-through): nodes carry an `Authority` attribute, which can be used to filter or weight.

This is not a semantic knowledge graph. There is no `Requirement(X) → Subject(Y)` ontology, no `OrdinaryCourseExceptionApplies(X)` predicate. Those live in Plane 4. Plane 3 is the structural skeleton; Plane 4 is the symbolic execution layer.

## Plane 4: Symbolic execution

The symbolic plane is a rule engine over a normalized fact table. Each clause that expresses a logical rule is compiled (on ingest) into a typed predicate. For § 547(b):

```
Avoidable(transfer T, debtor D, creditor C, asOf t) ⇐
    Beneficiary(T, C),
    AntecedentDebt(T, D),
    Insolvent(D, time(T)),
    OR(
        Within(time(T), filing(D), 90 days),
        AND(
            Within(time(T), filing(D), 365 days),
            Insider(C, D, time(T))
        )
    ),
    NOT(Excepted(T, asOf t)).
```

And for § 547(c)(9):

```
Excepted(transfer T, asOf t) ⇐
    NOT(ConsumerDebtor(debtor(T))),
    Amount(T) < Threshold547c9(t).

Threshold547c9(t) =
    $7,575 if t ∈ [2022-04-01, 2025-03-31]
    $6,825 if t ∈ [2019-04-01, 2022-03-31]
    ...   (from § 104 adjustment table)
```

These rules are compiled by a human (or a human-supervised LLM-assisted process) once per statute, version-tagged, and stored. At query time, if the parser identifies that the query carries facts (a transfer amount, a date, an insider/non-insider attribute) the symbolic plane evaluates the relevant rule and returns either a definite answer or a list of missing facts.

Formally, the symbolic plane is a function

$$
\text{Symbolic}(q, F, t) = \{\, (r, \text{verdict}, \text{trace}) \,:\, r \in R(q), \text{verdict} \in \{\text{true}, \text{false}, \text{undetermined}\} \,\}
$$

where $R(q)$ is the set of rules implicated by the query, $F$ is the fact set extracted from the query, and the trace is the satisfied (or unsatisfied) premise list with citations to the source clauses.

The symbolic plane fixes failure mode 5 (temporal and numeric reasoning) completely. It is also the cleanest way to handle failure mode 6 (confidence): a rule that evaluates definitively with all premises grounded is high-confidence; a rule with `undetermined` verdicts because facts are missing is, by construction, low-confidence in a way that triggers abstention.

The hard part of the symbolic plane is not the engine. Datalog, Soufflé, or a small bespoke evaluator works fine. The hard part is the rule library: deciding which clauses warrant a formal rule, writing the rule, versioning it, and keeping it in sync with statutory amendments. This is real human work, and the returns scale roughly with how heavily a clause is queried.

A practical heuristic: invest in formalizing the 10 to 20 most-queried rules in your corpus first. They will cover a disproportionate share of real queries. The long tail can be handled by retrieval-only paths with appropriate abstention.

## Plane 5: Score fusion

With four planes, candidate fusion is the next decision. Suppose each plane produces a candidate set $C_k$ with scores $s_k(d)$ for $d \in C_k$. We want a fused score $s(d)$ that combines them.

The simplest fusion is reciprocal rank fusion (RRF):

$$
\text{RRF}(d) = \sum_{k} \frac{1}{60 + \text{rank}_k(d)}
$$

RRF works when scores are not comparable across planes. It is rank-only and ignores score magnitudes. It is a strong default when labels are scarce.

A better fusion, when 100 to 200+ labeled query-document pairs are available, is a learned convex combination:

$$
s(d) = \sum_k \alpha_k \cdot \tilde{s}_k(d) + \sum_l \beta_l \cdot \phi_l(d, q)
$$

where $\tilde{s}_k$ is a min-max-normalized score from plane $k$, the $\phi_l$ are structural features (jurisdiction match indicator, effective-date validity, citation-graph path length to a seed, authority weight), and $(\alpha, \beta)$ are learned by logistic regression or LightGBM against labeled relevance judgments. With $\alpha_k \geq 0$ and $\sum_k \alpha_k = 1$, this is a convex combination over normalized scores; the structural features are pure additive lift.

The key theoretical result, due to Bruch et al., is that score-based hybrid fusion is sample-efficient: with surprisingly few labeled pairs (low hundreds), a learned fusion outperforms rank-only fusion, and the gain over each individual plane is monotone in the number of planes when the planes have low score correlation. Crucially, lexical, dense, graph, and symbolic plane scores have low pairwise correlation because they exploit different signal types, so adding planes adds genuine information rather than redundancy.

Geometrically, fusion is a projection from a 4-dimensional score space (plus structural features) onto a single scalar. The hyperplane is fit to maximize NDCG@k on labeled data. The lower the correlation between planes, the more degrees of freedom the projection has, and the more relevance signal the projection captures.

This is the mathematical reason structure-first retrieval dominates hybrid retrieval: a 4-plane fusion has strictly more degrees of freedom than a 2-plane one, and the planes added (graph, symbolic) carry signal that lexical and dense provably cannot. The dominance is not empirical hope; it follows from the score-correlation structure.

## Plane 6: Domain-aware reranking

The reranker operates on the top-$k$ fused candidates (typically $k = 100$ to $150$). A standard reranker scores

$$
s_{\text{rerank}}(q, d) = f_\theta(q, d)
$$

where $f_\theta$ is a cross-encoder. For structure-first retrieval we extend it with explicit features:

$$
s_{\text{rerank}}(q, d) = f_\theta(q, d) + \gamma_1 \cdot \mathbb{1}[\text{jurisdiction match}] + \gamma_2 \cdot \mathbb{1}[\text{date valid}] + \gamma_3 \cdot \text{authority weight} + \gamma_4 \cdot \text{citation path support} + \gamma_5 \cdot \mathbb{1}[\text{symbolic rule satisfied}] - \gamma_6 \cdot \text{contradiction count}
$$

The $\gamma$ coefficients are learned on a held-out set with reranker-only and reranker-plus-features ablations. In practice the structural features carry most of the precision lift on failure-mode-4 and failure-mode-5 queries (jurisdiction and temporal questions), while the neural reranker carries the lift on topical disambiguation.

A subtle point: the reranker's $f_\theta$ should be a strong cross-encoder (monoT5, MiniLM-L12, or a listwise model like ListT5), but it does not need to be domain-fine-tuned for the structure-first design to work. The structural features carry the domain knowledge; $f_\theta$ provides general topical discrimination. This is a budget-friendly result. Domain fine-tuning is expensive and brittle, while structural features are cheap and audit-friendly.

## Calibrated confidence

Confidence is a separate machine-learning problem from ranking. The reranker's logit is not a calibrated probability. To produce a calibrated confidence, train a post-hoc calibrator $g_\psi$ on features:

$$
\hat{p}(d \text{ correct} \mid q) = g_\psi(s_{\text{rerank}}(d), \text{margin}, \text{modality agreement}, \text{graph support}, \text{rule support}, \text{contradictions})
$$

where:
- `margin` is $s_{\text{rerank}}(d_1) - s_{\text{rerank}}(d_2)$ between the top and second candidate
- `modality agreement` is the fraction of planes that placed $d$ in their top-10
- `graph support` is the citation-path length from the query seed to $d$
- `rule support` is whether $d$ is cited as a premise of a satisfied symbolic rule
- `contradictions` is the count of retrieved passages with conflicting verdicts

The calibrator is fit by logistic regression or isotonic regression to minimize binary cross-entropy on labeled correct/incorrect pairs. Quality is measured by:

- **Expected Calibration Error (ECE):** $\text{ECE} = \sum_{b=1}^{B} \frac{|B_b|}{N} \| \text{acc}(B_b) - \text{conf}(B_b) \|$, where bins are formed by predicted confidence and `acc` is empirical accuracy within the bin.
- **Brier score:** $\text{Brier} = \frac{1}{N} \sum_{i=1}^{N} (\hat{p}_i - y_i)^2$, where $y_i \in \{0, 1\}$.
- **Reliability diagrams:** plot $\text{conf}(B_b)$ against $\text{acc}(B_b)$; on a well-calibrated model they lie on the diagonal.

Temperature scaling (the simplest calibration method) divides the logit by a learned scalar $T$ before softmax. It works well when the model is monotonically miscalibrated. For the multi-feature calibrator above, Platt scaling (logistic regression on the features) or isotonic regression are stronger choices and produce both calibrated confidence and a smooth abstention threshold.

The key theoretical observation: the calibrator's input features are not arbitrary. `Modality agreement` is high when the planes converge, which is empirically the strongest predictor of correctness. `Graph support` is high when the retrieved passage is reachable from the query's seed by a short typed walk. `Rule support` is true when a symbolic rule was satisfied. These features carry orthogonal information about *why* the candidate is correct, not just *how much* it scored. A confidence model trained on these features captures calibration in a way reranker-score temperature scaling cannot.

## Abstention as a precision lever

Abstention is the cheapest precision improvement available. Define:

$$
\text{decision}(q) =
\begin{cases}
\text{answer with } d^* & \text{if } \hat{p}(d^* \mid q) > \theta \\
\text{abstain} & \text{otherwise}
\end{cases}
$$

where $\theta$ is a threshold chosen to satisfy a target precision on the labeled set. Selective accuracy (accuracy conditioned on choosing to answer) rises monotonically with $\theta$, while coverage (fraction of queries answered) falls. The risk-coverage curve characterizes this tradeoff, and in legal applications a low-coverage, high-precision regime is almost always the right operating point.

A well-calibrated $\hat{p}$ is necessary for abstention to work in the right direction. If confidence is over-confident on the failure modes (as raw reranker score is), abstention will not trigger on the cases that need it. This is why calibration with structural features matters: those features expose failure modes that the reranker alone cannot.

A practical recipe: set $\theta$ so that on a labeled validation set, selective accuracy is 95% or higher, accepting whatever coverage that implies. Track coverage and selective accuracy in production. Re-calibrate after every meaningful index refresh.

## Why this dominates hybrid retrieval, formally

Let $R(q)$ denote the set of relevant documents for $q$ and $S(q)$ the set returned by a retrieval system. Recall is $|R \cap S| / |R|$ and precision is $|R \cap S| / |S|$.

A hybrid pipeline (BM25 + dense + reranker) returns

$$
S_{\text{hybrid}}(q) = \text{TopK}(f_{\theta}(q, d),\; \text{Cand}_{\text{BM25}}(q) \cup \text{Cand}_{\text{dense}}(q))
$$

The candidate set is the union of two text-based retrievals. Documents that match neither lexically nor semantically are absent, and the reranker cannot rescue them.

Structure-first retrieval returns

$$
S_{\text{struct}}(q) = \text{TopK}(g_\psi(\cdot),\; \text{Cand}_{\text{BM25}}(q) \cup \text{Cand}_{\text{dense}}(q) \cup \text{Cand}_{\text{graph}}(q) \cup \text{Cand}_{\text{symbolic}}(q))
$$

Recall is non-decreasing under structure-first retrieval because the candidate set is a superset:

$$
\text{Recall}_{\text{struct}}(q) \geq \text{Recall}_{\text{hybrid}}(q) \quad \forall q
$$

This is the trivial direction. The non-trivial direction is precision: adding more candidates can hurt precision if the reranker cannot discriminate well. But the structural and symbolic candidates carry explicit relevance evidence (citation-graph path, rule satisfaction) that the reranker's domain-aware extension uses. Empirically (and this is supported by the score-correlation analysis above) precision is also non-decreasing on legal queries because the planes contribute orthogonal signal.

More formally, let $X_k$ be the score from plane $k$ and $Y$ the binary relevance. The fusion model $\hat{Y} = h(X_1, \ldots, X_K)$ has minimum risk

$$
\inf_h \mathbb{E}[(Y - h(X))^2]
$$

bounded below by the conditional variance $\mathbb{V}[Y \mid X]$. Adding a plane shrinks $\mathbb{V}[Y \mid X]$ (strictly, when the plane carries information about $Y$ beyond the other planes). Graph support and symbolic verdicts carry such information for failure modes 1, 2, 4, 5, and 6. The irreducible risk is therefore strictly smaller under structure-first retrieval, and a well-trained fusion model attains lower expected error.

This argument is the scientific reason to prefer the four-plane architecture over hybrid retrieval. It is not "more components, more better." It is: the added components shrink the conditional variance of relevance given the score vector, by exploiting signals that text-text scoring cannot access.

## Training data and learning the fusion model

The architecture is only as good as the training and evaluation data behind its learned components. Three learned components matter: the dense encoder (or its fine-tuned legal variant), the fusion model $(\alpha, \beta, \gamma)$, and the confidence calibrator $g_\psi$. Each has its own data-construction discipline.

**Dense encoder adaptation.** Most legal teams start from a strong general encoder (BGE-M3, E5-large, or Cohere's embed-v4) and have between 50 and 500 labeled query-document pairs. That is far too few for from-scratch fine-tuning but enough to drive a meaningful adaptation. The recipe with the strongest empirical support combines three sources of supervision:

1. **GPL-style pseudo-labeling.** For each in-domain passage, generate $k$ synthetic queries with a query-generation model (T5-base fine-tuned on MS MARCO works well, as does a small Llama). Score each (query, positive, random-negative) triple with a strong cross-encoder; keep triples where the cross-encoder's positive-vs-negative margin exceeds a threshold. This produces $10^4$ to $10^5$ high-quality training triples from an unlabeled corpus. The resulting encoder typically gains 5 to 10 nDCG@10 over the base model on in-domain queries.

2. **Promptagator-style few-shot synthesis.** From your 50 to 500 real labeled pairs, sample 4 to 8 representative pairs as in-context examples and prompt an instruction-tuned LLM to generate synthetic queries for each unlabeled passage. The 4 to 8 example budget is enough to capture domain vocabulary and question style ("what is the minimum liability," "can the trustee avoid," "as of date X"). Filter aggressively with a reranker; expect to retain roughly 30% of generated queries.

3. **Hard-negative mining within structurally-similar passages.** For each (query, positive) pair, retrieve the top-50 candidates with a base encoder and select as hard negatives those passages that share the same jurisdiction, same source type, same effective year band, but are not the labeled positive. This forces the encoder to learn the *delta* between near-duplicates (exactly the discrimination the legal domain requires).

The encoder is trained with multiple-negatives ranking loss:

$$
\mathcal{L}_{\text{MNR}} = -\sum_i \log \frac{\exp(\cos(E(q_i), E(d_i^+)) / \tau)}{\sum_{j} \exp(\cos(E(q_i), E(d_j)) / \tau)}
$$

with temperature $\tau \approx 0.05$ and a batch size large enough that the in-batch negatives include several hard negatives per query.

**Fusion model.** The fusion model $(\alpha, \beta)$ learns to combine normalized scores from each plane plus structural features. With 100 to 500 labeled (query, document) relevance pairs, the data is too small to fit a complex model. LightGBM with 50 or fewer trees and small depth, or a logistic regression with L2 regularization, is the right complexity class.

Two construction principles. First, the training set must be stratified by query type. A test set dominated by "what is the minimum X" questions will train fusion weights that over-rely on lexical match and miss the gains on logical and temporal queries. Build 4 to 6 query-type strata (lookup, definition, exception application, threshold check, multi-clause synthesis, jurisdiction comparison) and sample roughly equally. Second, the labels must include not just binary relevance but a graded relevance (primary citation, supporting citation, background) so that fusion can learn to push the primary citation to position 1, not just into the top-10.

The fusion model's coefficients carry interpretable information: if $\beta_{\text{jurisdiction match}}$ is large, the model is learning that jurisdiction mismatch is a primary precision killer; if $\beta_{\text{rule support}}$ is large, the symbolic plane carries genuine signal. Track the coefficients over time; sudden shifts are a strong signal of distribution drift in the labeled set.

**Confidence calibrator.** The calibrator is trained on (features, correct-or-not) pairs derived from the fusion model's top-1 output. The "correct" label is whether the top-1 retrieved passage is a primary citation according to ground truth. With 500 labeled queries, the calibrator has 500 (binary, features) examples, adequate for isotonic regression or a small Platt scaler.

A subtle methodological point: the calibrator's training distribution must match the deployment distribution. If you train on queries sampled uniformly from your label set but deploy on user queries skewed toward a few high-frequency intents, the calibrator will be miscalibrated where it matters. Either weight the training samples by deployment query frequency, or build the labeled set by sampling from real query logs. The second approach is cleaner and is what production systems should do.

**Evaluation protocol.** Three points matter when evaluating this architecture.

First, report stratified metrics. A single nDCG@10 number averaged over all query types obscures exactly the dynamics that matter. Report nDCG, Recall@k, and Precision@5 separately for each query-type stratum, and report selective accuracy at a fixed coverage level (90% is a reasonable target) per stratum.

Second, control for version effects. If the test set was constructed against a snapshot of the corpus from time $t_0$ and the system is evaluated against a snapshot at $t_1$, all the version-drift failure modes will look like model errors when they are actually evaluation artifacts. Pin the corpus snapshot at evaluation time and report which version was used.

Third, measure abstention quality, not just answer quality. The risk-coverage curve (selective accuracy as a function of coverage) is the right summary of a retrieval system's safety properties. Two systems with the same overall accuracy can have very different risk-coverage curves; the one with the steeper decline at low coverage is the safer system.

A useful single-number summary is the area under the risk-coverage curve (AURC), defined as

$$
\text{AURC} = \int_0^1 \text{risk}(c) \, dc
$$

where $\text{risk}(c)$ is the error rate of the system when forced to answer the top $c$ fraction of queries (sorted by confidence). Lower AURC is better. For legal retrieval, AURC is more meaningful than accuracy because it captures the system's ability to know when not to answer.

**Inter-annotator agreement.** Legal labeling is harder than web-search labeling. Two competent attorneys may disagree about whether a particular passage is the "primary citation" for a particular query. Measure inter-annotator agreement with Cohen's $\kappa$ on a held-out set of 100 doubly-labeled queries. Expect $\kappa$ between 0.5 and 0.7, which is moderate to substantial agreement. A retrieval system that exceeds inter-annotator agreement on selective accuracy is, in a meaningful sense, performing at the upper bound of what is humanly achievable on those queries.

## Worked example: 11 U.S.C. § 547

Take the query from the previous post: *"A non-consumer debtor paid a $6,000 invoice to an outside vendor 100 days before filing. Can the trustee avoid this transfer under § 547?"*

**Query parsing.** The parser extracts:
- Jurisdiction: federal (no state mentioned, statute is § 547)
- As-of date: today (no historical date specified)
- Facts: `transfer_amount = $6,000`, `time_before_filing = 100 days`, `creditor_type = outside vendor (non-insider)`, `debtor_type = non-consumer`
- Predicate: "can the trustee avoid"
- Seed citation: § 547

**Plane 1 (lexical).** BM25 returns:
- § 547(b) (the rule)
- § 547(c)(2) (ordinary-course-of-business exception)
- § 547(c)(9) (small-transfer non-consumer exception)
- Several treatise sections discussing § 547

**Plane 2 (dense).** Returns paraphrases of (b), commentary on preference actions, and a few near-duplicates from different document versions.

**Plane 3 (structural graph).** Seeded at § 547(b), the traversal follows `cross_refs` to (c)(9) and (c)(2) — both are explicit exceptions named in (b)'s opening clause "Except as provided in subsections (c) and (i)." The traversal also walks `defines` edges to § 101(31) for "insider" and § 101(32) for "insolvent," providing definitional context. The effective-date filter (current) selects the post-2022 versions.

**Plane 4 (symbolic).** The fact set $F = \{T.\text{amount} = 6000, T.\text{days\_before\_filing} = 100, T.\text{creditor} = \text{non-insider}, T.\text{debtor} = \text{non-consumer}\}$ is evaluated against the rule library:

```
Avoidable(T) eval:
    Within(100, 0, 90 days)?           → false
    AND(Within(100, 0, 365), Insider)? → false (non-insider)
    OR(false, false)                   → false: (b)(4) unsatisfied
    Result: Avoidable(T) = false (independent ground 1)

Excepted(T) eval:
    NOT(ConsumerDebtor(D))?            → true (non-consumer)
    Amount(T) < $7,575?                → 6000 < 7575 → true
    Result: Excepted(T) = true (independent ground 2)
```

**Fusion.** All four planes converge on the same conclusion: the transfer is not avoidable. The fused candidate list places § 547(b)(4)(A), § 547(c)(9), and the rule trace at the top.

**Reranker.** Confirms ordering with high cross-encoder scores plus full structural-feature support: jurisdiction match (federal), date valid (current), authority (binding federal statute), citation path (b)→(c) intact, symbolic rule satisfied.

**Confidence.** Calibrator inputs: high reranker score, large margin to next candidate, modality agreement = 4/4, graph support = 1 (direct cross-reference), rule support = true, contradictions = 0. Predicted confidence ~0.97.

**Output.** The system answers "no, the trustee cannot avoid this transfer" with two independent grounds and full citation trace. Failure modes 1 through 6 are all eliminated: version-correct (1), exception retrieved (2), no paraphrase loss (3), authority pinned (4), arithmetic evaluated (5), confidence calibrated and supported by orthogonal evidence (6).

A hybrid pipeline on the same query, as shown in the previous post, returns § 547(b)(4)(A) and stops: confident, wrong-by-omission, and unaware of (c)(9).

## Worked example: Iowa Code § 321A.1

Take a different query: *"As of policy issuance on June 1, 2026, what is the Iowa statutory minimum bodily injury liability per person for a private passenger auto?"*

**Query parsing.** Extracts:
- Jurisdiction: Iowa
- As-of date: 2026-06-01
- Predicate: "what is the minimum"
- Coverage type: bodily injury per person, private passenger auto
- Seed citation: Iowa Code § 321A (motor vehicle financial responsibility)

**Plane 1 (lexical).** Returns multiple historical versions of § 321A.1, all matching "minimum bodily injury per person."

**Plane 2 (dense).** Returns paraphrases including underwriting commentary, NAIC model law references, and Iowa Department of Insurance bulletins.

**Plane 3 (structural graph).** Seeded at § 321A.1, the traversal follows `cross_refs` edges to definitional sections and to § 321A.21 (proof of financial responsibility). The `effective` filter set to 2026-06-01 retains only the version of § 321A.1 valid on that date, eliminating all superseded amounts. The structural plane has eliminated failure mode 1 by construction.

**Plane 4 (symbolic).** The rule for this query is a simple lookup of the version-tagged statutory amount; the symbolic plane returns the current per-person bodily injury minimum with the citation to the active version of § 321A.1 and the effective-date range.

**Fusion and rerank.** The current version dominates because every other plane agrees, the structural filter eliminated competing versions, and the symbolic plane returned a definitive lookup. Reranker confirms.

**Confidence.** All four planes agree, the symbolic verdict is definite, no contradiction. Confidence ~0.99.

**Output.** The system answers with the current Iowa per-person minimum, cites the active version of § 321A.1, marks the effective-date range, and notes any DOI bulletin that further restricts the minimum for specific vehicle classes (if such a bulletin is in the corpus and matched by structural traversal).

The hybrid baseline would have returned a passage from one of the historical versions with no way to know which version is current.

## Ablation design

To validate the architecture rigorously, the ablation suite should compare:

| Configuration | Lexical | Dense | Graph | Symbolic | Domain rerank | Calibrated abstention |
|---|---|---|---|---|---|---|
| H0: BM25 only | ✓ | | | | | |
| H1: BM25 + dense | ✓ | ✓ | | | | |
| H2: BM25 + dense + rerank | ✓ | ✓ | | | partial | |
| H3: + structural graph | ✓ | ✓ | ✓ | | partial | |
| H4: + symbolic | ✓ | ✓ | ✓ | ✓ | partial | |
| H5: + domain features in rerank | ✓ | ✓ | ✓ | ✓ | ✓ | |
| H6: full (with calibrated abstention) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

Metrics to track: Recall@k for k in {10, 50, 100}, Precision@5, NDCG@10, version-correct accuracy, jurisdiction-correct accuracy, evidence-completeness for multi-clause questions, ECE, Brier score, selective accuracy at 90% coverage, and coverage at 95% selective accuracy.

The expected pattern is that the structural graph (H3 over H2) is the single largest precision gain on legal corpora (typically twice the lift of going from dense alone to BM25+dense+rerank) because it eliminates failure modes 1, 2, and 4. The symbolic plane (H4 over H3) gives the largest lift on logical and temporal queries (failure mode 5). Domain features in reranking (H5 over H4) add modestly. Calibrated abstention (H6 over H5) shifts the operating point: selective accuracy rises sharply, coverage falls modestly.

The ablation is the canonical way to argue, in a publication or an internal review, that the four-plane design is doing real work rather than re-shuffling effort.

## Open questions

A few things remain genuinely open.

How do you scale the symbolic rule library without bottlenecking on human authoring? LLM-assisted rule extraction with human verification is the practical answer, but the verification cost is substantial and the failure-mode analysis of LLM-extracted rules is under-studied.

How should the calibrator handle distribution shift? Statutory amendments, new bulletins, and changing query patterns all shift the joint distribution of features and relevance. Re-calibration cadence and online adaptation are open engineering-and-science problems.

How do you measure relevance when ground truth itself is contested? Legal interpretation is sometimes genuinely ambiguous; the labeling protocol matters. Inter-annotator agreement on legal queries is lower than on web search queries, and benchmarks should report it.

How does this architecture interact with generative LLM-based answer composition? The retrieval system described here outputs an evidence bundle; the LLM is downstream. But the LLM can hallucinate even with perfect retrieval, and the joint system needs joint evaluation. The honest answer is: structure-first retrieval bounds the upstream contribution of error, but the downstream LLM is a separate failure surface.

## Closing

This is the architecture I built in response to the six failures in the previous post. The four planes (lexical, dense, structural graph, symbolic) carry orthogonal signal, and a learned fusion exploits that orthogonality. Calibrated confidence and abstention turn correctness into a tunable operating point. The math is not exotic. The engineering is real but bounded. The result is a retrieval system that, on legal corpora, dominates hybrid retrieval not by hope but by the structure of the conditional variance of relevance given the score vector.

The next post covers what it takes to actually build this in production: ingestion pipelines, storage choices, ANN index lifecycle, latency budgets per query stage, observability, and a deployment matrix that maps each component to cost-optimized and quality-optimized choices.
