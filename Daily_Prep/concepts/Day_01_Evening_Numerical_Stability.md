# C04 — Mixed Precision, Loss Spikes, and Numerical Stability

> **Day 1, Evening** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why this matters:** this is the "3am pager" genre of pretraining
> interview question, and it's where 18 months of real experience either shows or doesn't.
> The questions are diagnostic, not definitional: *"your loss spiked at step 41k. Walk me
> through it."*

---

## The One Idea

**Dynamic range and precision are separate resources, and every mixed-precision decision is
about which one you can afford to lose where.** Loss spikes are almost never random — they're
a quantity growing unbounded until it leaves the representable range of the format you chose.

---

## 1. The formats, and why BF16 won

| Format | Sign/Exp/Mantissa | Max value | Relative precision | Notes |
|---|---|---|---|---|
| FP32 | 1/8/23 | ~3.4e38 | ~7 decimal digits | The reference |
| **FP16** | 1/5/10 | **65,504** | ~3 decimal digits | More mantissa, **tiny range** |
| **BF16** | 1/8/7 | ~3.4e38 | ~2 decimal digits | FP32's exponent, truncated mantissa |
| FP8 E4M3 | 1/4/3 | 448 | ~1 digit | Forward pass / activations |
| FP8 E5M2 | 1/5/2 | 57,344 | <1 digit | Gradients — needs the range |

**The load-bearing insight:** BF16 has *the same exponent field as FP32*. Anything
representable in FP32 is representable in BF16, just coarsely. FP16 has more mantissa bits
but its range tops out at 65,504 and underflows around 6e-5 — and gradients live in exactly
that danger zone.

That's why FP16 training needs **loss scaling** (multiply the loss by ~2^15 before backward
to push gradients up out of the underflow region, unscale before the optimizer step, and
dynamically back off when you see an inf/NaN). BF16 needs none of that. **BF16 didn't win
because it's more accurate — it's less accurate. It won because it removed an entire class of
operational failure.**

If asked "why BF16 over FP16," the answer is dynamic range, not precision. Getting that
backwards is a tell.

---

## 2. Why FP32 master weights still exist

BF16 has ~7 mantissa bits, so around a weight of magnitude 1.0, the representable spacing is
roughly 2^-8 ≈ 0.004. A typical late-training update is `lr × normalized_grad` ≈ 1e-5.

```
w = 1.0, update = 1e-5  →  BF16(1.0 + 1e-5) = 1.0
```

**The update rounds to nothing and vanishes.** Not approximately — exactly zero progress. Late
in training, when the LR has decayed, *most* of your updates are below the rounding threshold.
Training silently stalls while the loss curve looks merely flat.

Hence: FP32 master weights, updated in FP32, cast down to BF16 for the forward/backward. This
is where most of the 16 bytes/param goes (C02).

**The alternative worth naming: stochastic rounding.** Round up or down with probability
proportional to distance, so small updates accumulate correctly *in expectation*. Lets you
drop the FP32 master copy and save 4 bytes/param. Increasingly common; a good thing to raise
unprompted.

---

## 3. Loss spikes: the actual causes

A spike is a symptom. Interviewers want your differential diagnosis, ranked.

| Cause | Signature | Fix |
|---|---|---|
| **Attention logit growth** | `q·k` magnitudes climb over training until softmax saturates; gradients vanish then explode. The most common cause in large runs. | **QK-LayerNorm** (normalize Q and K before the dot product). Near-free, extremely effective. |
| **Output logit drift** | Final logits drift to large absolute values; softmax denominator overflows | **z-loss**: an auxiliary penalty on `log²(Z)` keeping the partition function near 1 |
| **Bad data batch** | Sharp spike at one step, correlated with a specific shard. Repeated docs, corrupted encoding, a single pathological long document | Skip the batch, fix the shard, improve dedup. Log the data indices per step — without that you can't diagnose it at all. |
| **LR too high for current batch/schedule** | Spikes cluster right after warmup ends or after an LR bump | Lower peak LR, lengthen warmup |
| **Gradient clipping not actually engaging** | Grad-norm plot shows huge values passing through | Verify clipping is applied *after* all-reduce and *after* unscaling |
| **Optimizer state corruption** | Spike right after a checkpoint restart | Adam m/v not restored correctly; RNG/data-order state not restored |
| **Residual stream growth** | Activation norms grow with depth; later layers saturate | Scaled init (`1/√(2·n_layers)` on residual projections), sensible norm placement |

**Pre-LN vs Post-LN** is worth having a crisp opinion on: Post-LN gives better final quality
in principle but is notoriously unstable at depth without careful warmup; Pre-LN is far more
stable and is why essentially every large model uses it. Variants like sandwich norm exist to
recover some of Post-LN's quality.

---

## 4. The debugging protocol

When the loss spikes, "lower the LR and restart" is a *junior* answer. Show a process:

1. **Is it recoverable?** Many spikes self-heal in a few hundred steps. Don't intervene
   reflexively — you'll never learn the cause and you'll burn a restart.
2. **Instrument what you should already be logging:** per-layer grad norms, activation norms,
   attention logit max, optimizer state norms, and — critically — **the data indices for each
   step.** If you can't map step 41,203 back to exact samples, you cannot diagnose a data-driven
   spike. This is the single most common gap in real setups.
3. **Localize it.** Is the grad-norm explosion in one layer or global? One layer → architecture
   or init. Global → LR, data, or a numerics bug.
4. **Bisect data vs optimization.** Rewind to the last good checkpoint and replay *the same
   batches*. Spike reproduces → it's the data. It doesn't → it's stochastic/optimization.
5. **Then intervene**, in increasing order of cost: skip the batch → rewind and skip the shard
   → lower LR → change architecture (QK-norm, z-loss) → restart from scratch.

**"Rewind and replay the same batches" is the answer that separates people who have actually
done this from people who have read about it.**

---

## 5. Adjacent things that come up

- **FP8 training** — E4M3 forward, E5M2 backward, with per-tensor (or finer) scaling factors
  that must be tracked and updated. Roughly 2x throughput on Hopper+; the cost is
  operational complexity and a real risk of silent quality loss, so you need tight eval.
- **muP (maximal update parametrization)** — parametrize so optimal LR is invariant to width.
  Tune on a small proxy model, transfer to the big one. Saves enormous amounts of compute; a
  strong thing to mention when asked "how did you pick the LR for a model you can only train
  once?"
- **Determinism** — full bitwise determinism costs throughput (deterministic kernels, fixed
  reduction order). Most labs accept non-determinism and instead invest in *reproducible data
  order* plus frequent checkpointing. Know the tradeoff.
- **Silent data corruption** — at 1000+ GPU scale, a GPU computing wrong answers without
  erroring is a real, documented failure mode. Detection: periodic checksums, replica
  cross-checks, watching for one rank's grad norms diverging.

---

## Self-check

1. Why does FP16 need loss scaling and BF16 not? Answer in bits.
2. Show why BF16 weights without an FP32 master copy stall late in training. Use numbers.
3. Your loss spikes at step 41k. Give your first five diagnostic steps, in order.
4. What is QK-LayerNorm and which specific failure does it prevent?
5. What does z-loss regularize, and why does that keep softmax numerically safe?
6. You can only afford to train the large model once. How do you choose the learning rate?
