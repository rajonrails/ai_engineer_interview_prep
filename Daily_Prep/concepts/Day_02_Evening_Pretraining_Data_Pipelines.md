# C08 — Pretraining Data: Dedup, Filtering, Mixing, Annealing

> **Day 2, Evening** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** data work is the highest-leverage work in pretraining
> and the most under-prepared interview topic. Candidates rehearse parallelism and optimizers,
> then get asked "walk me through how you'd build a 15T-token corpus" and produce three
> sentences about Common Crawl.

---

## The One Idea

**At a fixed compute budget, the corpus decides the model.** Architecture choices move the
loss curve a little; data quality moves it a lot. Two teams with identical 70B architectures
and identical FLOPs will produce very differently capable models based entirely on what they
trained on. This is why the data team is not a support function.

Corollary for interviews: when asked how to improve a model and your compute is fixed, **data
is the first answer**, not a bigger model.

---

## 1. The pipeline

```
raw crawl → extract → language ID → dedup → quality filter → decontaminate
          → PII scrub → tokenize → mix/weight → shard → curriculum/anneal
```

Attrition is brutal and worth knowing as a sanity check: raw web crawl to trained tokens is
typically a **10-100x reduction**. If your pipeline keeps most of Common Crawl, your filters
aren't working.

---

## 2. Deduplication — the highest-value single step

Dedup reliably improves quality *and* reduces compute. It is the least glamorous and most
consistently valuable thing in the pipeline.

| Level | Method | Notes |
|---|---|---|
| **Exact document** | Hash the normalized text | Trivial, catches a surprising amount |
| **Near-duplicate document** | **MinHash + LSH**, Jaccard threshold ~0.8 | The workhorse. LSH makes it tractable at trillion-token scale by only comparing candidates that share a band. |
| **Substring** | Suffix array, remove repeated spans over ~50 tokens | Catches boilerplate, navigation chrome, licence blocks embedded in otherwise-unique docs |
| **Cross-split** | Dedup train against eval | This is decontamination — see below |

**Why it matters beyond compute savings:** duplicated text is effectively upweighted training
data you didn't choose to upweight, it encourages memorization over generalization, and
duplicated sequences are a documented driver of verbatim regurgitation.

---

## 3. Quality filtering

Three families, used together:

**Heuristic rules** (cheap, high volume) — Gopher-style: document length bounds, mean word
length, symbol-to-word ratio, fraction of lines ending in punctuation, fraction of duplicate
lines, stopword presence. Individually crude, collectively effective.

**Classifier-based** — train a fast classifier (fastText is the standard cheap choice) to
distinguish crawl text from a high-quality reference (Wikipedia, books, curated web), then
keep documents above a threshold. **Caveat worth raising:** this bakes in the reference
corpus's biases about what "good text" looks like, which can quietly narrow the distribution
— dialects, informal registers, and non-Western sources get filtered disproportionately.

**Perplexity filtering** — score with a small model trained on high-quality text, drop the
tails. Drop *both* tails: very high perplexity is garbage, very low perplexity is often
boilerplate or repetition.

**The judgment call interviewers probe:** aggressive filtering raises average quality but
shrinks the corpus and narrows diversity. When unique tokens are the binding constraint —
which they now are at frontier scale — over-filtering costs you more than it gains.

---

## 4. Decontamination

Eval contamination invalidates your benchmarks and, worse, you may not find out until after
you've made decisions based on them.

- Standard approach: **n-gram overlap** (commonly 13-gram) between training documents and
  every eval set, removing matches.
- Must run **after** dedup (or you'll miss contaminated near-duplicates) and must cover every
  benchmark you intend to report.
- **Partial contamination is the hard case**: a forum post quoting three MMLU questions. Exact
  match misses it; aggressive fuzzy matching deletes legitimate data.
- Honest position to take in an interview: full decontamination is not achievable at web scale.
  What you can do is measure and disclose it, and keep a private held-out eval that has never
  been published anywhere — that's the one you actually trust.

---

## 5. Mixing and weighting

Your corpus is domains — web, code, books, academic papers, multilingual, math, curated
reference — and the mixture is a hyperparameter with real consequences.

- **Code improves reasoning on non-code tasks.** Well-replicated and a good thing to cite.
- **Upsampling small high-quality sets** (books, academic text) is standard, but you're
  choosing repetition, which interacts with the epoch limits below.
- **Learned mixing (DoReMi-style)** — train a small proxy to find weights that minimize
  worst-case loss across domains, then transfer those weights to the big run. The principled
  alternative to hand-tuning.
- Mixture is tunable at small scale and transfers reasonably. Like LR, tune on a proxy.

---

## 6. Repetition, annealing, and synthetic data

**Epochs.** Unique high-quality text is now a binding constraint, so repetition is unavoidable.
Rough consensus: **up to ~4 epochs behaves close to fresh data**; beyond that returns decay
sharply toward zero. Plan the corpus around that.

**Annealing / midtraining.** In roughly the final 10% of training, upweight the highest-quality
data and decay the LR toward zero. This reliably produces gains disproportionate to the compute
spent, and it's one of the most practically useful things to know — the *order* of data matters,
not just its composition. Expect to be asked why: late-training updates are small and
targeted, so what you show the model last disproportionately shapes where it lands.

**Synthetic data.** Increasingly load-bearing (textbook-style generated content, rephrased web
text, generated code with execution-verified correctness). The failure mode to name is **model
collapse** — training on unfiltered generations of a previous model narrows the distribution
over successive rounds. Mitigations: keep a strong real-data anchor, and verify synthetic data
externally (execute the code, check the proof) rather than trusting the generator.

---

## 7. Two details that catch people out

- **Tokenizer interacts with data.** Vocabulary size trades sequence length against embedding
  and output-projection size. A bigger vocab means fewer tokens for the same text — cheaper
  training per unit of content — but a larger embedding matrix and a more expensive softmax.
  Multilingual corpora need larger vocabularies or non-English text tokenizes terribly
  (sometimes 3-4x more tokens per word), which is both a cost and a quality problem. The
  tokenizer must be trained on a sample matching your final mixture.
- **Resumable data loading.** If a run dies at step 40,000, you must resume at *exactly* the
  right sample — not just the right step. Many pipelines get this wrong and silently re-show
  or skip data. It's also a prerequisite for the "replay the same batches" loss-spike
  diagnostic from C04.

---

## Self-check

1. Walk through building a 15T-token corpus. Name every stage and what it removes.
2. Why does MinHash+LSH scale when pairwise Jaccard doesn't?
3. Why drop *both* perplexity tails?
4. What is annealing/midtraining and why does it beat uniform data ordering?
5. You have 3T unique high-quality tokens and need 15T. Options, with trade-offs?
6. What's model collapse and how do you avoid it while still using synthetic data?
