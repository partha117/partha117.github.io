---
title: "Why Hybrid Retrieval Fails on Legal Corpora"
date: 2026-04-19
permalink: /posts/2026/why-hybrid-retrieval-fails/
description: "Six specific failure modes of BM25 + dense + reranker pipelines on legal text — version drift, missing exceptions, load-bearing words, jurisdiction bleed, temporal arithmetic, and false confidence — illustrated on 11 U.S.C. § 547."
tags:
  - retrieval
  - RAG
  - hybrid-search
  - legal-AI
---
I built hybrid retrieval for a legal and insurance corpus at work. The standard pipeline: BM25 for lexical matching, a dense encoder for semantics, RRF to fuse the two candidate sets, a cross-encoder reranker on top. This is the default architecture in 2026. It dominates BEIR. I was confident.

In production, it returned wrong answers. Confidently. With no surface signal that anything was off.

This post is about the six failure modes I cataloged while rebuilding the system. They all come out of one real statute, 11 U.S.C. § 547, which is the kind of text that breaks retrieval systems in ways that only show up when a real lawyer catches the answer.

## The pipeline

![Hybrid retrieval pipeline: query branches into BM25 and a dense encoder, both feed into RRF fusion, then a cross-encoder reranker, producing a final ranked list.](/images/hybrid-retrieval-pipeline.svg)
*Standard three-stage hybrid retrieval pipeline.*

Each stage has a job. BM25 catches statutory phrases, citation tokens, exact numbers. The dense encoder catches paraphrase and vocabulary mismatch. The reranker re-scores the top hundred candidates by reading query and passage together. On standard benchmarks this pipeline is very good. On legal corpora it has six specific blind spots.

## The running example

Section 547 of the U.S. Bankruptcy Code lets a trustee claw back payments a debtor made shortly before filing. It includes an exception in subsection (c)(9): transfers below a dollar threshold cannot be avoided. That threshold is indexed every three years under § 104. As of April 2022 it is $7,575. The April 2025 adjustment may have changed it.

That ambiguity is load-bearing. Everything below follows from it.

## Failure mode 1: Version drift

**What the query looked like.** "For a Chapter 7 case filed in May 2026 by a non-consumer debtor, what is the minimum transfer value below which the trustee cannot avoid under § 547(c)(9)?"

**What the pipeline returned.** The $7,575 passage. Confidently. Correct citation, correct subsection.

**Why it was wrong.** A realistic legal corpus contains the pre-2022 version ($6,825), the 2022 version ($7,575), and possibly a post-2025 version. BM25 matches all three on "§ 547(c)(9)" and "non-consumer." The dense encoder matches all three semantically. The reranker scores all three highly because they are all topically correct. The correct answer depends on an effective-date filter the retriever does not have.

```
Corpus contains:
  [v2019]  § 547(c)(9): less than $6,825   ← BM25 score: high
  [v2022]  § 547(c)(9): less than $7,575   ← BM25 score: high
  [v2025?] § 547(c)(9): less than $????    ← BM25 score: high

Reranker sees three high-scoring candidates.
Reranker has no access to effective dates.
Reranker returns one. User trusts it.
```

This is the most common failure mode in legal retrieval and the most dangerous because there is no surface signal of trouble.

## Failure mode 2: The exception you never see

**What the query looked like.** "A non-consumer debtor paid a $6,000 invoice to an outside vendor 100 days before filing. Can the trustee avoid this transfer?"

**What the pipeline returned.** The 90-day rule from (b)(4)(A): "the trustee may avoid any transfer made on or within 90 days before the date of the filing."

**Why it was wrong.** The transfer is 100 days out, which puts it outside the 90-day window. Even if it were inside, subsection (c)(9) excepts non-consumer transfers under $7,575 anyway. The correct answer is no, with two independent reasons. The pipeline got neither.

The chunk boundary is the proximate cause. The subsection (b) rule and the subsection (c)(9) exception were stored in separate chunks. BM25 surfaced (b)(4)(A) because the query mentioned "trustee avoid transfer." The exception was never a candidate. The reranker cannot fix that because it only reranks what the fan-out gave it.

Exceptions in legal text are almost always narrower and less cited than the rules they qualify. They are statistically invisible to retrieval systems that use popularity signals. The feature that makes them legally important, that they are the special case, is exactly what makes them easy to miss.

## Failure mode 3: Paraphrase loses the load-bearing word

**What the query looked like.** "Is the trustee required to avoid a preferential transfer, or does the trustee have discretion?"

**What the pipeline returned.** Passages paraphrasing § 547 as "the trustee will pursue preferential transfers" or "preferential transfers are clawed back."

**Why it was wrong.** The actual text of § 547(b) says "the trustee *may* avoid." May. Not shall. That word is the entire answer: discretion, not duty. Dense embeddings treat "shall" and "may" as nearly synonymous in vector space. A cross-encoder trained on the general internet has been trained to be paraphrase-tolerant. Paraphrase tolerance is the wrong inductive bias when the question pivots on a modal verb.

This failure shows up rarely in standard benchmarks because most queries are topical. It shows up regularly in legal practice because obligation questions are exactly what lawyers ask.

## Failure mode 4: Jurisdiction bleed

**What the query looked like.** "Under New York law, is § 547(c)(9)'s threshold different from the federal threshold?"

**What the pipeline returned.** Passages from New York federal district court opinions interpreting § 547.

**Why it was wrong.** Section 547 is federal law. There is no New York version. The correct answer is that the question does not quite parse: § 547 applies uniformly under federal bankruptcy jurisdiction and states do not modify it. The system returned New York-flavored federal passages and an LLM summarized them as if they were New York-specific law.

The retriever has no model of authority hierarchy. Federal statute, state regulation, circuit court opinion, state supreme court ruling: all are passages. To the retriever they are equally weighted text. Which passage is binding on whom is not encoded anywhere in the index.

## Failure mode 5: Temporal arithmetic does not live in embeddings

**What the query looked like.** "A debtor's payment to an insider creditor was made 200 days before the bankruptcy filing. Is this transfer avoidable under § 547?"

**What the pipeline returned.** The 90-day rule and the one-year insider rule. Both relevant passages. Both ranked correctly.

**Why it was incomplete.** The answer requires checking: 200 > 90 (so (b)(4)(A) does not apply) and 200 < 365 (so the insider extension in (b)(4)(B) does apply). That is arithmetic over temporal thresholds. No part of the pipeline does arithmetic. The reranker surfaces the right rule but cannot apply it to the specific fact.

In production, users are one mental calculation from the answer and that step is where mistakes happen, especially on boundary cases and under time pressure.

## Failure mode 6: Reranker confidence is not answer correctness

Every failure above produced a high-confidence reranker score. That is the point. Reranker scores are calibrated against topicality, not correctness. A cross-encoder that has never seen a § 547 version-drift case will assign a high score to the pre-2022 passage because it is topically correct. High topical score, wrong answer, no signal to the user.

The practical consequence is that building abstention on top of reranker confidence produces the wrong behavior. The system will abstain on rare-vocabulary queries where lexical match is weak, and it will not abstain on version-drift cases where lexical match is strong but to the wrong version. Exactly the wrong inversion.

## The same pattern in insurance law

Iowa Code § 321A.1 sets minimum financial responsibility limits for motor vehicle operators. The amounts have been amended several times. A retriever asked "what is the Iowa minimum bodily injury per person?" surfaces the 1995 version, the 2010 version, and the current version equally. Failure mode 1.

State DOI bulletins reinterpret statutory minimums. Internal underwriting manuals overlay carrier-specific rules on top of the regulatory floor. A question like "what minimum should I write for a commercial education service vehicle in Iowa under our current underwriting guide?" requires resolving four layers, each with its own effective date, supersession rule, and authority weight. Hybrid retrieval flattens all four into passages about Iowa minimums.

The other five failure modes recur in the same form. Version drift, missing exceptions, load-bearing words lost in paraphrase, jurisdiction bleed across state lines, date arithmetic on policy lapse windows, reranker confidence inversions. The failure modes are structural, not specific to bankruptcy law.

## Why adding more reranker does not help

I tried fine-tuning the reranker on legal pairs. It improved relative ranking within a topic. It did not fix any of the six failure modes.

Version drift requires effective-date metadata, not better ranking. The missing exception requires that the exception be in the candidate pool, which requires following cross-reference edges at retrieval time. Load-bearing words require a retriever that is not paraphrase-tolerant for those words. Authority hierarchy requires explicit authority fields, not cosine similarity. Temporal arithmetic requires a rule engine, not a transformer. Confidence calibration is a separate problem from ranking.

A better reranker improves the ordering of candidates within a topic. It does not add structure the candidates lack, retrieve passages that were not fetched, or perform reasoning the architecture does not support.

**The fix is structural.** Parse documents into a hierarchy with first-class fields: jurisdiction, effective date, authority weight, citation graph, defined terms, numeric thresholds. Index those fields as hard filters. Follow cross-reference edges at retrieval time. Run a small rule engine over normalized facts when the query is logical or numeric. Then layer hybrid retrieval and reranking on top of that structured substrate.

That is what I built next.
