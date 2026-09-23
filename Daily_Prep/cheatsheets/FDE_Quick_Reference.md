# FDE Quick Reference

> Everything numerical and quotable from C01, C03, C05, C07, C09, C11, C13, C15.
> Built for the 30 minutes before an onsite. Each line links back to the full concept.

---

## The numbers

| Quantity | Formula / value |
|---|---|
| Forward-pass FLOPs | `2 × N_params × N_tokens` |
| Training FLOPs | `6 × N × D` (2 fwd + 4 bwd) |
| KV cache per token | `2 × n_layers × n_kv_heads × head_dim × bytes` |
| Llama-3-70B KV | ~**320 KB/token** (GQA, 8 KV heads) → 8K ctx ≈ 2.6 GB/sequence |
| Same model without GQA | ~2.6 MB/token — **8x worse** |
| Decode arithmetic intensity, batch 1 | **1 FLOP/byte** vs H100's ~150 needed → <1% of compute used |
| Eval 95% CI half-width | `≈ 0.98/√n` → n=20 is **±22pp**, n=400 is ±5pp |
| Total latency | `TTFT + (output_tokens × TPOT)` |
| LoRA trainable params | `r × (d_in + d_out)` → r=16, d=4096 ≈ **0.8%** of full FT |

---

## The one-liners to actually say

- **"Inference is two workloads: prefill is compute-bound, decode is memory-bandwidth-bound."** They have opposite fixes. — [C01](../concepts/Day_01_Morning_Inference_Economics.md)
- **"At batch size 1 you're using under 1% of the GPU."** Batching is the whole game; your batch ceiling is set by KV cache, which is set by context length. — C01
- **"Output tokens cost ~3x because of a hardware asymmetry, not pricing strategy."** — C01
- **"82% → 86% on n=100 is not a result."** The CIs overlap almost entirely. — [C03](../concepts/Day_01_Evening_Evals_And_Measurement.md)
- **"Use a paired design."** Same examples, both variants, McNemar — detects the same effect with ~an order of magnitude fewer samples. — C03
- **"Prompts are not a security boundary."** A system prompt saying *max 10 recipients* is a suggestion; a function that raises on #11 is a control. — [C05](../concepts/Day_02_Morning_Agents_And_Action_Safety.md)
- **"You cannot prompt your way out of prompt injection."** The mitigation is capability restriction. — C05
- **"Retrieval recall@k is a hard ceiling on everything downstream."** — [C07](../concepts/Day_02_Evening_RAG_Retrieval_Quality.md)
- **"If the correct answer were pasted into the prompt, would it get it right?"** Yes → retrieval. No → maybe fine-tuning. — [C09](../concepts/Day_03_Morning_Fine_Tuning_Decisions.md)
- **"Fine-tuning changes behavior, not knowledge — and it's a commitment, not an experiment."** — C09
- **"LLM systems fail silently. Nothing throws."** — [C11](../concepts/Day_03_Evening_Production_Monitoring.md)
- **"The stated problem is a hypothesis, not a spec."** — [C13](../concepts/Day_04_Morning_Customer_Scoping.md)
- **"I'd rather find out in two weeks that this doesn't work than in six months."** — C13
- **"The context window is a budget with a quality gradient, not a bucket."** — [C15](../concepts/Day_04_Evening_Context_Engineering.md)

---

## Diagnostic ladders

**"It's too slow"** → split `TTFT` vs `TPOT` first.
TTFT bad → prompt length, no prefix cache, prefill queueing. TPOT bad → memory bandwidth: smaller model, quantization, speculative decoding. Both fine but total bad → too many output tokens. **Always check p99 vs p50 separately — tails generate complaints, medians don't.**

**"It's too expensive"**, ranked by typical impact:
1. Don't call the model (exact/semantic cache)
2. Prefix caching — often 50%+ when there's a stable system prompt
3. Route by difficulty (small model for the easy 80%)
4. Cut output tokens (~3x price)
5. Cut input tokens (better retrieval, lower top-k, drop stale few-shots)
6. Quantize / self-host — only at sustained volume with full batches

**"RAG isn't working"**, in order:
1. **recall@k** — can the answer be found at all? Everything else is premature.
2. Rank position — found but ranked 40th → reranker, not prompt
3. Read the retrieved chunks yourself — unglamorous, finds the most bugs
4. Faithfulness — right context, wrong answer → generation
5. **Only now** touch the prompt

**"It got worse last Tuesday"**:
Establish the delta (change log) → replay the golden set → decompose retrieval vs generation → **roll back first, diagnose second** → check the boring things (provider status, index age, expired creds) → write the eval row.

---

## Framework tables

**Action safety tiers** — [C05]

| Tier | Example | Controls |
|---|---|---|
| 0 read-only | lookup, search | scoped credential, audit log |
| 1 reversible write | draft, create ticket | audit, undo, rate limit |
| 2 irreversible internal | delete, refund | confirmation gate, idempotency key, caps, dry-run |
| 3 irreversible external | email customers, move money | human in loop, hard caps, staged rollout, kill switch |

**Blast radius** = `auto-approve ceiling × fleet rate limit`. Must hold when the model is *captured*, not just buggy.

**The fine-tuning ladder** — [C09]: prompt → few-shot → retrieval → context/tools → fine-tune. Fine-tune is rung 5. The answer is never "never," it's *"why did rungs 1–4 fail?"*

**Eval ladder** — [C03]: assertions (free, CI) → LLM-judge (cents, nightly) → human review (calibrates the judge) → online A/B.
Judge biases: position (run both orderings), verbosity, self-preference (different model family), rubric drift (pin the version), score compression (prefer pairwise).

---

## Techniques by the problem they solve

| Problem | Technique |
|---|---|
| GPU idle while short requests wait | Continuous batching (10–20x throughput) |
| KV cache fragmentation | PagedAttention |
| Re-prefilling a stable prefix | **Prompt/prefix caching** — usually the single biggest cost lever |
| Decode wastes compute | Speculative decoding (2–3x, identical output distribution) |
| Long prefill blocks everyone's decode | Chunked prefill |
| Weight-read bandwidth + footprint | Quantization (FP8/INT8 cheap; 4-bit has real cost) |
| Vector search misses `E4021` | Hybrid BM25 + vector, fused with **RRF over ranks, not scores** |
| Right doc, ranked 40th | Cross-encoder reranking — highest ROI single addition |
| Precise match vs. enough context | Small-to-big chunking |
| Redundant top-k | MMR |
| Turn-30 context bloat | **Structured memory** > summarization > sliding window |

---

## Traps

- Proposing architecture before asking questions — the most common scoping failure.
- Accepting "we need to fine-tune" at face value.
- Reporting accuracy without a confidence interval.
- Aggregate metrics with no slicing — a 3pp drop is usually one slice off a cliff.
- Putting a safety limit in the prompt.
- Variable content (timestamp, username) in the cached prefix.
- Forgetting that a demo is 10% of the work and 90% of the perceived progress.
