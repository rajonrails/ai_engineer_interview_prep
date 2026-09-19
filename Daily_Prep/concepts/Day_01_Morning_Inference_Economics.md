# C01 — Inference Economics: Prefill, Decode, and the KV Cache

> **Day 1, Morning** · Track: FDE (primary) + Pretraining (shared foundation) · ⭐⭐⭐⭐
> **Read time:** ~20 min. **Why this first:** ~40% of FDE technical conversations are
> secretly this conversation. "It's too slow," "it's too expensive," "can we use a bigger
> model," "why is the first token slow but the rest fast" — all one topic.

---

## The One Idea

**LLM inference is two completely different workloads wearing one trenchcoat.**

Almost every wrong answer about latency and cost comes from treating inference as a single
thing. It isn't. It's a *compute-bound* phase followed by a *memory-bandwidth-bound* phase,
and they have opposite optimization strategies.

| | **Prefill** | **Decode** |
|---|---|---|
| What it does | Process the whole prompt at once | Generate one token at a time |
| Parallelism | All prompt tokens in parallel | Strictly sequential — token N needs token N-1 |
| Bottleneck | **Compute** (FLOPs) | **Memory bandwidth** (HBM reads) |
| User-visible metric | **TTFT** — time to first token | **TPOT** — time per output token |
| Scales with | Prompt length (quadratically in attention, linearly in FFN) | Output length |
| Batching helps? | Not much — already saturating compute | **Enormously** — this is the whole game |
| Cost per token | Cheap (~1/3 the price of output tokens, roughly) | Expensive |

That last row is why every API provider charges more for output tokens than input tokens.
It isn't arbitrary pricing — it reflects a real hardware asymmetry. **If you can explain
that asymmetry in an interview, you're already above the median candidate.**

---

## 1. Prefill: compute-bound

You hand the model a 10,000-token prompt. Every token can be processed simultaneously —
it's one big matrix multiply per layer, exactly like a training forward pass.

Rule of thumb for a dense transformer forward pass:

```
FLOPs ≈ 2 × N_params × N_tokens
```

(The 2 is one multiply + one add per parameter. Memorize this — it's the basis of the
training-compute formula `C ≈ 6ND` too, where the 6 is 2 forward + 4 backward.)

**Worked example — 70B model, 10K-token prompt:**

```
FLOPs = 2 × 70e9 × 10e3 = 1.4e15 = 1.4 PFLOP
```

An H100 does roughly 500 TFLOPS dense BF16. At a realistic ~50% utilization that's
250 TFLOPS effective:

```
1.4e15 / 2.5e14 = 5.6 seconds  (on one GPU)
```

Split 8 ways with tensor parallelism: **~0.7s TTFT.** That is your first-token latency, and
it's dominated by arithmetic. The GPU is genuinely busy. You cannot batch your way out of it.

> **Nuance interviewers probe:** attention is O(n²) in sequence length while the FFN is O(n).
> At short contexts the FFN dominates and prefill is ~linear in prompt length. At long
> contexts the quadratic attention term takes over. So "doubling my prompt doubled my TTFT"
> is true at 2K tokens and false at 100K.

---

## 2. Decode: memory-bandwidth-bound

Now the model generates. Token 501 cannot start until token 500 exists. No parallelism across
time. For **each single token**, the GPU must stream **every weight** from HBM into the compute
units.

**Same 70B model, batch size 1:**

- Weights to read: 70e9 params × 2 bytes (BF16) = **140 GB, per token**
- H100 HBM bandwidth: ~3.35 TB/s
- Across 8 GPUs: 17.5 GB each ÷ 3.35 TB/s ≈ **5.2 ms/token** → ~190 tok/s ceiling

Now the punchline. Compute done for that token: `2 × 70e9 = 140 GFLOPs`. Bytes moved: 140 GB.

```
Arithmetic intensity = 140e9 FLOPs / 140e9 bytes = 1 FLOP per byte
```

An H100 needs roughly **150 FLOPs per byte** to saturate its tensor cores. At batch size 1
you are running the GPU at **under 1% of its compute capability.** You bought a supercomputer
and used it as a very expensive memcpy.

**The fix is batching.** The weights you stream in are the *same weights* for every sequence
in the batch. Read once, use 128 times:

```
Batch size 128 → arithmetic intensity ≈ 128 FLOPs/byte → near roofline
```

Throughput goes up ~100x. Per-user latency barely moves. **This is why self-hosting a model
for low traffic is economic suicide and why API providers are cheap: they batch your request
with a thousand strangers'.** If you take one thing from today, take this.

---

## 3. The KV cache: what makes it complicated

To avoid recomputing attention over the entire history for every new token, we cache the
Key and Value tensors for every previous token. That cache is the reason decode is fast —
and the reason you run out of memory.

```
KV bytes per token = 2 (K and V) × n_layers × n_kv_heads × head_dim × bytes_per_element
```

**Llama-3-70B** (80 layers, 8 KV heads via GQA, head_dim 128, BF16):

```
2 × 80 × 8 × 128 × 2 = 327,680 bytes ≈ 320 KB per token
```

- One 8K-token conversation: **~2.6 GB**
- 100 concurrent users at 8K context: **~260 GB** — more than 3 full H100s, *on top of* the
  140 GB of weights

Note `n_kv_heads = 8`, not 64. That's **Grouped Query Attention**: multiple query heads share
one KV head. Without GQA (i.e., 64 KV heads) that same model would need **2.6 MB/token** —
8x more — and serving 100 users would need over 2 TB of KV cache. GQA is not a minor
optimization; it's what makes long-context serving economically possible at all.

**This is the central tension in LLM serving:**

> Memory is a fixed budget split between weights and KV cache. Bigger batches → better
> throughput → lower cost. But bigger batches and longer contexts both eat KV cache. Your
> max batch size is set by your KV cache budget, and your KV cache budget is set by context
> length. **Context length and throughput are in direct competition for the same HBM.**

A candidate who says "we'll just increase the batch size" without mentioning KV cache memory
gets marked down. A candidate who says "what's the p95 context length? that sets our batch
ceiling" gets marked up.

---

## 4. The techniques worth naming

These come up by name constantly. Know what problem each one solves.

| Technique | Solves | How |
|---|---|---|
| **Continuous batching** (vLLM, TGI) | GPU idles while short requests wait for long ones to finish | Evict finished sequences and admit new ones every decode *step*, not every batch. Often 10–20x throughput over naive static batching. |
| **PagedAttention** | KV cache fragmentation — you must pre-allocate for worst-case length and waste most of it | Virtual-memory-style paging for KV blocks. Enables much higher real batch sizes. |
| **Prompt / prefix caching** | Re-prefilling the same 5K system prompt on every request | Cache the KV for a shared prefix. Huge win for agents and chat, where the prefix is stable. Often the single biggest cost lever in RAG. |
| **Speculative decoding** | Decode is bandwidth-bound and wastes compute | A small draft model proposes k tokens; the big model *verifies* all k in one pass (that pass is compute-bound, which is free capacity). 2–3x speedup, mathematically identical output distribution. |
| **Chunked prefill** | A 100K-token prefill blocks everyone else's decode (head-of-line blocking) | Slice the prefill and interleave it with decode steps. Trades slight TTFT for much better p99 TPOT. |
| **Quantization** (FP8, INT8, AWQ/GPTQ) | Both weight-read bandwidth *and* memory footprint | Halving bytes/param ~halves decode time and frees HBM for more KV cache. Quality cost is small at 8-bit, real at 4-bit. |
| **GQA / MQA** | KV cache size | Share KV heads across query heads. Architectural — decided at pretraining time, not something you bolt on. |

---

## 5. How this shows up as an interview answer

When someone says "it's too slow," the first move is **always** to split the metric:

```
Total latency = TTFT + (output_tokens × TPOT)
```

These have *disjoint* fixes. Optimizing the wrong one wastes weeks.

- **TTFT is bad** → prompt is too long, or no prefix caching, or prefill is queued behind
  other requests. Fix: shrink the prompt, cache the prefix, chunked prefill, more compute.
- **TPOT is bad** → memory bandwidth. Fix: smaller model, quantization, speculative decoding,
  better hardware. Batching will *not* help one user's TPOT.
- **Total is bad but both look fine** → you're generating too many tokens. Fix: constrain
  output format, stop generating prose around JSON, stream so perceived latency drops.

And on cost, the levers ranked by typical real-world impact:

1. **Don't call the model** — cache exact/semantic duplicates. Free.
2. **Prefix caching** — if you have a stable system prompt or RAG preamble, this is often 50%+.
3. **Route by difficulty** — small model for the 80% of easy queries, escalate the rest.
4. **Cut output tokens** — they cost ~3x input. "Respond only with JSON" is a cost optimization.
5. **Cut input tokens** — better retrieval, top-k=5 not top-k=50, drop the few-shot examples
   the model no longer needs.
6. **Quantize / self-host** — only worth it at sustained high volume where you can keep
   batches full.

---

## Self-check (answer before tonight's drop)

1. Why do providers charge ~3x more for output tokens than input tokens? Answer in terms of
   hardware, not pricing strategy.
2. Your TPOT is 40ms and you need 20ms. Someone suggests doubling the batch size. What
   happens to your TPOT, and why?
3. A customer wants 200K context *and* high throughput on fixed hardware. Why can't they
   have both, and what's the formula that proves it?
4. When does speculative decoding *not* help? (Hint: what is it exploiting?)

---

**Tonight:** evals — how to actually measure whether an LLM system got better, and why
"we eyeballed 20 examples" is the most common reason AI projects die.
