# Pretraining Drill Bank

> Rapid-fire self-test. Cover the answer, say yours out loud, then check.
> Mark anything you fumble and re-drill it in two days.

---

## Parallelism and memory

**Q. Derive the 16 bytes/param figure. Now redo it for 8-bit Adam.**
BF16 weights 2 + BF16 grads 2 + FP32 master 4 + m 4 + v 4 = 16. With 8-bit Adam, m and v drop to 1 each → **10 bytes/param**.

**Q. Why is TP capped around 8 in practice?**
It issues two *blocking* all-reduces per transformer layer per microbatch, sized by activations — orders of magnitude more frequent than DDP's once-per-step, and unoverlappable. NVLink absorbs it; InfiniBand does not. TP=8 is the node boundary, not a convention.

**Q. PP=8 with a 30% bubble. Microbatch count, and cheapest fix?**
`(8−1)/(m+7) = 0.30` → m ≈ 16. Cheapest fix is more microbatches (m=64 → 10%), then interleaved 1F1B. Check the resulting global batch against critical batch size.

**Q. FSDP moves 3N bytes vs DDP's 2N, yet is often faster on large models. How?**
DDP requires the full model per rank, forcing smaller batches or not fitting at all. FSDP's comm is spread across the step and prefetchable behind compute, so much of it hides. The comparison isn't bytes, it's wall clock under a memory constraint.

**Q. Rank by comm volume per step: TP, PP, DDP, ZeRO-3. Now by latency sensitivity. Why do they differ?**
Volume: PP < DDP(2N) < ZeRO-3(3N) < TP (activation-sized × per layer). Latency sensitivity: TP highest (blocking, per layer), then ZeRO-3, then DDP, then PP. Volume is bytes; sensitivity is about blocking and frequency.

**Q. 400B on 512 H100s. Show it fits.**
Params+grads (4 B/param) sharded by TP×PP=64 → 25 GB. Optimizer (12 B/param) sharded by 512 → 9.4 GB. Total 34.4 GB, leaving ~45 GB for activations. **Without ZeRO-1 across DP it's 100 GB/rank and doesn't fit.**

---

## Numerics

**Q. Why does FP16 need loss scaling and BF16 not? Answer in bits.**
FP16 has 5 exponent bits → max 65,504, underflow ~6e-5, and gradients live in that danger zone. BF16 has 8 exponent bits — the same range as FP32 — so nothing underflows; it pays in mantissa (7 bits) instead.

**Q. Show why BF16 weights without an FP32 master copy stall late in training.**
Around w=1.0, BF16 spacing is ~2⁻⁸ ≈ 0.004. A late-training update of ~1e-5 rounds to exactly zero. Progress stops while the loss curve merely looks flat. (Alternative: stochastic rounding.)

**Q. What is QK-LayerNorm and which failure does it prevent?**
Normalizing Q and K before the dot product. Prevents attention-logit growth — the most common cause of loss spikes in large runs — where `q·k` magnitudes climb until softmax saturates, gradients vanish, then explode.

**Q. What does z-loss regularize and why does that help?**
It penalizes `log²(Z)` on the softmax partition function, keeping Z near 1 so output logits don't drift to magnitudes that overflow.

**Q. Loss spikes at step 41k. First five diagnostic steps.**
1) Is it recoverable, or did it plateau above trend. 2) Per-layer grad norms, activation norms, attention logit max, and **data indices**. 3) Localize: one layer or global. 4) **Replay identical batches from the last checkpoint** to bisect data vs optimization. 5) Intervene cheapest-first: skip shard → lower LR → architecture.

---

## Scaling and data

**Q. 3e23 FLOPs. Derive N and D, then GPU-days on 512 H100s at 40% MFU.**
`C=120N²` → `N=√(3e23/120)≈5e10` (50B), `D≈1T`. Compute: 3e23 / (512 × 2e14) ≈ 2.9e6 s ≈ **34 days**.

**Q. Why is Chinchilla the wrong target for a model you'll serve widely?**
It minimizes loss per *training* FLOP and ignores inference. Total cost ≈ `6·N·D_train + 2·N·D_inference`. At volume the second term dominates, pushing toward smaller models trained far longer — Llama-3-8B at ~1,875 tokens/param.

**Q. 10x compute but only 1.5x more unique tokens. What do you do?**
Repeat up to ~4 epochs (close to fresh-data value), improve filtering to raise quality within what you have, add verified synthetic data, and skew the remaining compute toward parameters — while watching that you don't exceed critical batch size.

**Q. Why does MinHash+LSH scale when pairwise Jaccard doesn't?**
Pairwise is O(n²) — impossible at trillion-token scale. MinHash compresses each document to a signature; LSH bands signatures so only documents sharing a band are compared, making it near-linear.

**Q. Why drop *both* perplexity tails?**
High perplexity is garbage. Very low perplexity is boilerplate, templates, and repetition — technically well-formed, informationally empty.

**Q. What is annealing and why does it beat uniform ordering?**
In the last ~10%, upweight the highest-quality data and decay the LR toward zero. Late updates are small and targeted, so what the model sees last disproportionately shapes where it lands. Data *order* matters, not just composition.

---

## Attention and optimizers

**Q. KV cache/token: 32 layers, 32 heads, 8 KV heads, head_dim 128, BF16.**
`2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KB/token`.

**Q. Why is FlashAttention faster while doing *more* FLOPs?**
The bottleneck was HBM traffic, not arithmetic. Tiling plus online softmax means the n×n score matrix is never written to HBM; backward recomputes scores rather than storing them. More compute, far less memory movement. Exact, not approximate.

**Q. Why divide by √d_head?**
`q·k` is a sum of `d_head` products, so its variance grows with dimension. Unscaled, logits grow, softmax saturates toward one-hot, gradients vanish.

**Q. What does YaRN do that Position Interpolation doesn't?**
PI compresses *all* RoPE frequencies, degrading fine-grained local resolution. YaRN is frequency-dependent: leaves high-frequency (local) dimensions alone, interpolates low-frequency (long-range) ones, plus an attention-temperature correction.

**Q. Why does evicting the first tokens from a sliding window collapse quality?**
Attention sinks. Softmax must sum to 1, so heads with nothing relevant to attend to dump mass on the first tokens. Remove them and that mass redistributes onto real content, corrupting the distribution.

**Q. Why AdamW rather than Adam + L2?**
L2 added to the gradient gets divided by `√v̂` along with everything else, so high-gradient parameters receive *less* decay — the opposite of the intent. AdamW applies `λ·θ` directly to the weight update, decoupled from adaptive scaling.

**Q. Why β₂ = 0.95 rather than 0.999?**
0.999 averages the second moment over ~1000 steps — too sluggish at scale, and a single bad batch persists in `v` for a long time.

**Q. Where must gradient clipping happen in a distributed step?**
After all-reduce and after unscaling. Clip before the all-reduce and each rank clips its *local* gradient, which is a different operation and silently changes the update.

**Q. Batch size 4x. How do you change the LR for Adam, and why not linear?**
Roughly ×2 (√4). Linear scaling is the SGD rule; Adam's per-parameter normalization already absorbs part of the gradient-noise reduction, so linear over-scales.

**Q. Explain muP in three sentences.**
Optimal LR normally shifts with model width, so hyperparameters tuned on a small model are wrong on a large one. muP re-parametrizes init scale, per-layer LRs, and output multipliers as functions of width so that optimal hyperparameters become width-invariant. You sweep on a cheap proxy and transfer directly to the run you can only do once.

---

## Fault tolerance and post-training

**Q. 400B, 2048 GPUs, 8-min checkpoint, 6-hour MTBF. Optimal interval and expected loss?**
`τ = √(2 × 0.133h × 6h) = √1.6 ≈ 1.26 h` ≈ every 75 min. Expected loss per failure ≈ half that, ~38 min, plus restart overhead.

**Q. Four things a checkpoint must contain beyond weights and optimizer state.**
Per-rank RNG state · **exact data-loader position** (not step count) · LR scheduler state and step · the full config including code version and parallelism layout.

**Q. Why is a straggler worse than a dead node?**
A dead rank triggers a timeout and an automated restart. A slow-but-alive rank silently throttles all 1024, because collectives run at the speed of the slowest participant, and nothing alerts.

**Q. Why does the KL penalty exist in RLHF? What happens at β=0?**
It keeps the policy near the SFT reference. At β=0 the policy drifts to exploit the reward model — reward hacking: verbosity inflation, sycophancy, formatting tics that score well and mean nothing.

**Q. DPO's core idea in two sentences, and what it gives up.**
The optimal policy under the RLHF objective has a closed form in the reward, so you invert it and express reward implicitly through the policy — then optimize directly on preference pairs. You give up a reward model, so you cannot score fresh samples, and it's off-policy.

**Q. Four things post-training needs from your base model.**
Latent capability (code and math in the mixture lift reasoning broadly) · calibration · format diversity · a distribution not over-narrowed by aggressive filtering. Plus a reproducible checkpoint with the exact tokenizer.
