# C07 — RAG and Retrieval Quality

> **Day 2, Evening** · Track: FDE · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** RAG is still the most common thing an FDE deploys, and
> "our RAG isn't working" is the most common thing a customer says. The interview tests whether
> you debug it systematically or reach for the prompt.

---

## The One Idea

**Retrieval recall is a hard ceiling on everything downstream. If the answer-bearing chunk
isn't in the context, no prompt, no model upgrade, and no reranker can recover it.**

The default failure pattern in the field: recall@10 is 60%, so 40% of questions are
unanswerable by construction, and the team spends three weeks tuning the generation prompt.
**Measure recall first** — it's the diagnostic that reorders the entire work plan, and saying
so is what separates a systematic answer from a list of techniques.

---

## 1. The pipeline and where each stage fails

```
query → [transform] → [retrieve: BM25 + vector] → [fuse] → [rerank] → [assemble] → generate
```

| Stage | Failure | Symptom |
|---|---|---|
| **Chunking** | Answer split across a boundary; chunk lacks context to be independently meaningful | Retrieval finds the right *document*, wrong *piece* |
| **Embedding** | Domain vocabulary out of distribution; asymmetric query/doc lengths | Semantically-close-but-wrong chunks rank high |
| **Retrieval** | Pure vector misses exact identifiers, part numbers, error codes, names | "What does error E4021 mean" returns general troubleshooting prose |
| **Ranking** | Right chunk retrieved at position 40, top_k=5 | Recall@50 fine, recall@5 terrible — a *ranking* problem, not retrieval |
| **Assembly** | Relevant chunk buried mid-context | "Lost in the middle" — models attend less reliably to middle content |
| **Generation** | Model ignores context, or over-trusts contradictory sources | Hallucination *despite* good retrieval |

Each has a different fix. Conflating them is the mistake.

---

## 2. Chunking

| Strategy | When |
|---|---|
| **Fixed-size + overlap** (e.g. 512 tokens, 50 overlap) | Baseline. Overlap exists to survive boundary splits. |
| **Structural** (by heading, section, function) | Whenever the document has real structure. Almost always better than fixed-size. |
| **Semantic** (split where embedding similarity drops) | Prose without structure. Expensive; modest gains. |
| **Contextual retrieval** | Prepend an LLM-generated summary of the parent document to each chunk before embedding. Substantially improves recall — the chunk carries its own context instead of relying on the reader to infer it. Costs a one-time generation pass over the corpus. |
| **Parent-document / small-to-big** | Embed small chunks for precise matching, but return the enclosing section for generation. Best of both; usually the right default. |

**The trade-off to state out loud:** small chunks retrieve precisely but lack context; large
chunks carry context but dilute the embedding and waste tokens. Small-to-big exists because
you shouldn't have to choose.

---

## 3. Hybrid search — and why keyword search is not obsolete

Dense vectors capture meaning; BM25 captures *exact lexical match*. They fail differently,
which is why combining them beats either.

**BM25 still wins on:** product codes and SKUs, error codes, people's names, acronyms, API
method names, rare technical terms, and anything the embedding model never saw in training.
A customer searching "E4021" wants documents containing the literal string `E4021`, and an
embedding model has no idea that token is special.

**Fusing them — Reciprocal Rank Fusion:**

```
RRF(d) = Σ  1 / (k + rank_i(d))        k ≈ 60 by convention
```

The point of RRF is that it uses **ranks, not scores** — so you never have to normalize a
cosine similarity against a BM25 score, which are on incomparable scales. That's the reason
it's the default, and it's a good detail to know.

**MMR (Maximal Marginal Relevance)** solves a different problem: redundancy. If your top 5
chunks are near-duplicates, you've spent 5 slots on one fact. MMR trades relevance for
diversity via `λ·relevance − (1−λ)·max_similarity_to_selected`. Matters for
multi-document-synthesis questions.

---

## 4. Reranking — the highest ROI single addition

Retrieve broadly (top 50-100) with cheap methods, then rerank with a **cross-encoder** that
sees query and document *together* and scores their interaction directly. Bi-encoders (normal
embeddings) must compress each document into a vector before ever seeing the query;
cross-encoders don't, so they're far more accurate — and far too slow to run over the whole
corpus. Hence the two-stage design.

Typical: retrieve 100 → rerank → keep top 5. Often the single biggest quality jump available,
and cheap to add.

---

## 5. Query-side transformations

The user's query is frequently a bad search string.

| Technique | What it does | When |
|---|---|---|
| **Multi-query** | Generate 3-5 paraphrases, retrieve for each, fuse | Vocabulary mismatch between users and docs |
| **HyDE** | Have the model write a *hypothetical answer*, embed that, search with it | Short queries; answer-shaped text embeds closer to answer-shaped documents |
| **Decomposition** | Split multi-hop questions into sub-questions, retrieve per sub-question | "How does our refund policy differ from the 2023 version?" |
| **Metadata extraction** | Pull filters out of the query (date, product, tenant) and use them as hard constraints | Nearly always worth it. A date filter beats any amount of semantic cleverness. |
| **Step-back** | Ask a more general question first to get grounding context | Narrow questions that need background |

---

## 6. RAG vs. long context

With 1M-token windows, "just put everything in the prompt" is a real option — and a real
interview question.

**Long context wins:** small corpora (under ~100K tokens), questions needing whole-document
reasoning, prototypes where retrieval infrastructure isn't worth it yet.

**RAG still wins on:** cost (you pay for every token of that 1M window, every call —
revisit C01), latency (prefill is compute-bound and 1M tokens is a lot of prefill),
corpora too large to fit regardless, access control (you can't filter by permission if you
dumped everything in), attribution and citation, and freshness (update an index, not a cache).

**The real answer is usually both:** retrieve aggressively into a large window. You don't need
top-5 precision when you can afford top-50 — long context makes RAG *more* forgiving, not
obsolete.

---

## 7. Debugging in order

1. **Recall@k.** Can the answer even be found? Build a small labeled set: question → the chunk
   that answers it. Everything else is premature until this is good.
2. **Rank position.** Found but ranked low → reranker.
3. **Chunk quality.** Read the retrieved chunks yourself. Are they coherent, self-contained?
   This unglamorous step finds more bugs than any metric.
4. **Faithfulness.** Given the right context, does the answer use it? If not, that's generation:
   prompt, citation requirements, or model.
5. **Only now** touch the generation prompt.

---

## Self-check

1. Why is recall@k a ceiling on end-to-end RAG accuracy?
2. Customer searches "E4021" and gets generic troubleshooting docs. Diagnose and fix.
3. Why does RRF use ranks rather than scores?
4. Explain small-to-big chunking and the trade-off it resolves.
5. Customer has a 2M-token corpus and a 1M-token model. Do you still build RAG? Argue both sides.
6. Recall@50 is 94% but recall@5 is 61%. What do you build?
