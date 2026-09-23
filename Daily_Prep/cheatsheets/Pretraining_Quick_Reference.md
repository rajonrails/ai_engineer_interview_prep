# Pretraining Quick Reference

> Everything numerical and quotable from C02, C04, C06, C08, C10, C12, C14, C16.
> Built for the 30 minutes before an onsite.

---

## The numbers you must have cold

| Quantity | Formula / value |
|---|---|
| Training compute | `C ≈ 6ND` (N params, D tokens) |
| Mixed-precision AdamW state | **16 bytes/param** — BF16 w(2) + BF16 g(2) + FP32 master(4) + m(4) + v(4) |
| 70B model state | 1.12 TB before a single activation |
| Chinchilla-optimal | `D ≈ 20N`; for fixed C, `N ∝ C^0.5`, `D ∝ C^0.5` |
| Chinchilla at 10²⁴ FLOPs | `N = √(C/120) ≈ 91B`, `D ≈ 1.8T` |
| Pipeline bubble | `(p−1)/(m+p−1)` — p=8,m=64 → 10% |
| Optimal checkpoint interval | **Young/Daly: `τ ≈ √(2δM)`** — 70B/1024 GPUs ≈ 46 min |
| Expected loss per failure | half the checkpoint interval |
| Comm volume per step | DDP `2N` · ZeRO-3/FSDP `3N` · TP activation-sized **per layer, blocking** · PP p2p only |
| BF16 | 1/8/7 — FP32's exponent, 7 mantissa bits |
| FP16 | 1/5/10 — more mantissa, **max 65,504** |
| MFU target | 35–50%; below 30% something is wrong |

---

## The one-liners to actually say

- **"Every parallelism trades memory for communication. Map each one onto the interconnect tier that matches its communication intensity."** — [C02](../concepts/Day_01_Morning_Parallelism_Strategies.md)
- **"TP stops at 8 because it crosses the node boundary"** — per-layer *blocking* all-reduces need NVLink, not because 16 "doesn't work." — C02
- **"BF16 won on dynamic range, not precision."** It is strictly *less* accurate than FP16; it removed loss scaling, an entire class of operational failure. — [C04](../concepts/Day_01_Evening_Numerical_Stability.md)
- **"Without FP32 master weights, a 1e-5 update rounds to exactly zero against BF16's ~0.004 spacing."** Training silently stalls while the curve just looks flat. — C04
- **"Chinchilla minimizes loss per *training* FLOP and ignores inference entirely."** Llama-3-8B ran ~1,875 tokens/param — 94x past the ratio — deliberately. — [C06](../concepts/Day_02_Morning_Scaling_Laws.md)
- **"At a fixed compute budget, the corpus decides the model."** — [C08](../concepts/Day_02_Evening_Pretraining_Data_Pipelines.md)
- **"FlashAttention does more FLOPs and is much faster, because the bottleneck was never FLOPs."** Exact, not approximate. — [C10](../concepts/Day_03_Morning_Attention_And_Long_Context.md)
- **"Context length is not context capability."** Needle tests are near-saturated and measure retrieval, not reasoning. — C10
- **"At scale, failure is the steady state, not an exception path."** The question is what fraction of compute you lose. — [C12](../concepts/Day_03_Evening_Checkpointing_And_Fault_Tolerance.md)
- **"A dead rank hangs the job, it doesn't crash it."** Collectives block until timeout. — C12
- **"A straggler is worse than a dead node."** Collectives run at the speed of the slowest participant. — C12
- **"Pretraining decides what the model *can* do; post-training decides what it *does*."** — [C16](../concepts/Day_04_Evening_Post_Training_Handoff.md)

---

## Parallelism selection

| Strategy | Memory | Comm | Place it |
|---|---|---|---|
| **DDP** | none | `2N` all-reduce, once/step, overlappable | outermost |
| **ZeRO-1/2/3 (FSDP)** | `16/d` at stage 3 | `3N`, spread through the step, prefetchable | within/across nodes; HSDP avoids the cliff |
| **TP** | shards matrices | activation-sized, **per layer, blocking** | **inside a node only, ≤8** |
| **PP** | shards layers | p2p at stage boundaries — lowest volume | **across nodes** |
| **EP (MoE)** | shards experts | all-to-all ×2 per layer, routing-sensitive | as the topology allows |

Standard recipe: `TP=8 (NVLink) × PP=k (IB) × DP/FSDP (rest)`, plus SP/CP for long context.
**Check the layout fits**: 400B on 512 GPUs needs ZeRO-1 across DP or per-rank state is 100 GB > 80 GB.

---

## Loss spikes — ranked causes

| Cause | Signature | Fix |
|---|---|---|
| **Attention logit growth** | gradual ramp; `q·k` climbing over training | **QK-LayerNorm** |
| Output logit drift | softmax denominator overflow | **z-loss** |
| Bad data shard | sharp, correlates with specific indices | skip/fix shard; *requires per-step data-index logging* |
| LR too high | clusters after warmup or an LR bump | lower peak, longer warmup |
| Clipping not engaging | huge grad norms passing through | clip **after** all-reduce and **after** unscaling |
| Optimizer state corruption | spike right after a restart | m/v or RNG/data state not restored |
| Residual stream growth | activation norms grow with depth | scaled init `1/√(2·n_layers)` |

**Protocol:** is it recoverable → instrument (per-layer grad norms, attention logit max, **data indices**) → localize (one layer or global) → **replay identical batches** to bisect data vs optimization → intervene cheapest-first.

*Full recovery = transient. Plateau above trend = it didn't self-heal; intervene.*

---

## Debugging an MFU drop

1. One rank or all? → straggler vs systemic
2. Periodic, matching checkpoint interval? → I/O blocking
3. Data loader starving the GPUs?
4. Comm still overlapping? (an FSDP all-gather that stopped prefetching)
5. Workload changed? (longer seqs → attention's quadratic share; padding ratio)
6. Hardware throttling? (clocks, power caps, temps)

---

## Data pipeline

`raw → extract → lang ID → dedup → quality filter → decontaminate → PII → tokenize → mix → shard → anneal`
Raw-to-trained attrition is typically **10–100x**. Keeping most of Common Crawl means your filters don't work.

- **Dedup:** exact hash → **MinHash+LSH** (Jaccard ~0.8) → substring (suffix array) → cross-split
- **Filters:** heuristic (Gopher rules) + classifier (fastText vs reference — *narrows the distribution*) + perplexity (**drop both tails**)
- **Decontamination:** 13-gram overlap, after dedup. Not fully achievable at web scale — keep a private never-published eval.
- **Repetition:** ~**4 epochs** behaves close to fresh; beyond that returns decay sharply
- **Annealing:** last ~10%, upweight quality + decay LR. Order matters, not just composition.
- **Model collapse:** unfiltered synthetic recursion narrows the distribution — keep a real-data anchor, verify externally (execute the code)

---

## Optimizer and schedule

```
m = β₁m + (1−β₁)g ;  v = β₂v + (1−β₂)g²
m̂ = m/(1−β₁ᵗ) ;  v̂ = v/(1−β₂ᵗ)
θ ← θ − lr·( m̂/(√v̂+ε) + λθ )     ← decoupled decay = the W in AdamW
```

- **Adam+L2 ≠ AdamW:** L2 in the gradient gets divided by `√v̂`, so high-gradient params get *less* decay — backwards. Don't decay biases/LayerNorm gains.
- **β₂ = 0.95**, not the 0.999 default — 1000-step averaging is too sluggish at scale.
- **Warmup** protects the window where `v̂` is unreliable — same window bias correction protects.
- **Cosine** needs total steps up front. **WSD** lets you stop anywhere in the stable phase, and pairs with data annealing.
- **LR vs batch:** `√batch` for Adam (linear is the SGD rule). Past critical batch size, nothing helps.
- **muP** = the answer to *"you only get one run"*: width-invariant optimal HPs, sweep on a proxy, transfer.

---

## Attention and context

- **MHA → MQA → GQA:** GQA settled it — near-MHA quality, most of MQA's cache savings. **A pretraining-time choice that fixes your serving cost forever.**
- **`√d_head`** exists because dot-product variance grows with `d_head`; without it softmax saturates and gradients vanish.
- **FlashAttention:** tiling + online softmax, never materializes the n×n matrix. `O(n²) → O(n)` memory, exact.
- **RoPE:** `θ_i = base^(−2i/d)`. Rotation difference depends only on `m−n`, so relative position falls out of the dot product. Low i = high freq = local.
- **Context extension:** PI (compresses all frequencies) < NTK (raise base) < **YaRN** (frequency-dependent — leave high-freq local resolution alone). Then brief long-sequence fine-tuning.
- **Attention sinks:** softmax must sum to 1, so heads dump spare mass on the first tokens. Evict them from a sliding window and quality collapses.

---

## Post-training interface

`pretrain → SFT → preference (RLHF/DPO) → RLVR → eval/safety`

- **RLHF:** reward model + PPO with `− β·KL(π‖π_ref)`. **β=0 → reward hacking** (verbosity, sycophancy).
- **DPO:** invert the closed-form optimal policy, optimize preference pairs directly. No reward model — so no way to score fresh samples, and it's off-policy.
- **RLVR:** verifier as reward where correctness is checkable. Cleanest signal; limited to where a verifier exists.
- **What post-training needs from you:** latent capability (code+math in the mixture lift reasoning broadly), calibration, format diversity, a corpus not over-narrowed by filtering, a reproducible checkpoint with the exact tokenizer.
- **Alignment tax:** track base-model evals through the whole pipeline, not just chat evals.
