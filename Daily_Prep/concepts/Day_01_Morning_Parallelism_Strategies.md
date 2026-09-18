# C02 — Choosing a Parallelism Strategy: DP, FSDP, TP, PP, EP

> **Day 1, Morning** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why this first:** this is *the* pretraining interview topic. With
> 18 months in, you've likely used a config someone else wrote. The interview asks you to
> *derive* it — "here's a 400B model, 512 H100s, 32K context. What's your parallelism plan,
> and defend every number." Assume you'll be pushed on communication volume, not vocabulary.

---

## The One Idea

**Every parallelism strategy trades memory for communication. Pick by asking which one you're
actually short on, and place each cut where the interconnect can afford it.**

Junior answer: "we used FSDP." Senior answer: "TP=8 inside the node because those all-reduces
are per-layer and need NVLink, PP=4 across nodes because point-to-point is the cheapest thing
you can put on InfiniBand, DP for the rest, and here's the bubble math."

---

## 1. First, the memory budget — this drives everything

Mixed-precision training with AdamW, per parameter:

| Item | Precision | Bytes/param |
|---|---|---|
| Weights | BF16 | 2 |
| Gradients | BF16 | 2 |
| Master weights | FP32 | 4 |
| Adam momentum (m) | FP32 | 4 |
| Adam variance (v) | FP32 | 4 |
| **Total** | | **~16 bytes/param** |

**This is the number to have memorized.** A 70B model needs **~1.12 TB** of state before a
single activation exists. On 80 GB H100s, the model state alone needs 14 GPUs. Not "we'd
prefer to shard" — arithmetically cannot fit.

Then activations on top, which scale with `batch × seq_len × hidden × layers` and, for
naive attention, `batch × heads × seq²`. At long context, **activations dominate model state.**
That's the whole reason sequence/context parallelism exists.

Two levers before you shard anything:
- **Activation checkpointing (recompute)** — store layer boundaries only, recompute the rest
  in backward. Cuts activation memory a lot; costs ~30% extra compute. Selective/"full" vs
  per-layer granularity is a real tuning knob, not a boolean.
- **Optimizer choice** — 8-bit Adam or a shampoo/muon-family optimizer changes the 16 bytes.

---

## 2. The five cuts

### Data Parallel (DDP) — replicate the model, split the batch
- **Memory:** no help at all. Every rank holds the full 16 bytes/param.
- **Comm:** one gradient all-reduce per step, ~2N bytes. Once per step, overlappable with
  backward. Cheapest possible comm pattern.
- **Use when:** the model fits on one device. For frontier pretraining it never does, so DDP
  is the *outer* loop, never the whole answer.

### FSDP / ZeRO — shard the model state across data-parallel ranks
| Stage | Shards | Memory/rank |
|---|---|---|
| ZeRO-1 | optimizer states | 4 + 12/d |
| ZeRO-2 | + gradients | 2 + 14/d |
| ZeRO-3 (= FSDP full-shard) | + parameters | 16/d |

- **Comm:** all-gather params in forward (N), all-gather again in backward (N),
  reduce-scatter gradients (N) ≈ **3N vs DDP's 2N** — 1.5x, and it's spread across the step
  rather than batched at the end. Good implementations prefetch the next layer's all-gather
  behind the current layer's compute, which hides most of it *if* your bandwidth is adequate.
- **Use when:** you need memory relief and have decent interconnect. It's the default modern
  baseline and it's simple — no model surgery.
- **The catch:** comm scales with the *number of shards*. Push FSDP across too many nodes and
  you fall off a bandwidth cliff. Hybrid sharding (shard within a node, replicate across) is
  the standard fix — HSDP.

### Tensor Parallel (TP) — split individual matrices across GPUs
- Split attention heads and FFN weight matrices column/row-wise. Megatron's trick: column-split
  then row-split so you need exactly **two all-reduces per transformer layer per forward**
  (one after attention, one after MLP), and two more in backward.
- **Comm:** proportional to *activations* (`batch × seq × hidden`), and it happens
  **per layer, per microbatch** — orders of magnitude more frequent than DDP's once-per-step.
  It is also **blocking**: you cannot overlap it, the next op needs the result.
- **Therefore: TP stays inside a node.** NVLink (~900 GB/s) can absorb it; InfiniBand
  (~50–400 GB/s) cannot. **TP ≤ 8 is the near-universal rule**, and "why not TP=16?" is a
  standard interview trap. The answer is "because it crosses the node boundary," not
  "because it doesn't work."
- **Sequence parallel** is TP's companion: it additionally splits the LayerNorm/dropout
  regions along sequence, cutting activation memory further for near-free.

### Pipeline Parallel (PP) — split layers into sequential stages
- **Comm:** only point-to-point activation sends at stage boundaries. **Lowest volume of any
  strategy** — which is exactly why it's the one you send across slow inter-node links.
- **The cost is the bubble.** With `p` stages and `m` microbatches:

```
bubble fraction = (p - 1) / (m + p - 1)
```

  p=4, m=8 → 27% of your GPUs idle. p=4, m=64 → 4.5%. **So PP demands many microbatches,
  which demands a large global batch size.** Interleaved 1F1B scheduling (multiple
  non-contiguous stages per device) shrinks the bubble further at the cost of more comm.
- **The other cost:** load balancing. The last stage holds the LM head + loss (huge vocab
  projection), the first holds embeddings. Naive equal-layer splits leave stages idle.

### Expert Parallel (EP) — for MoE, spread experts across devices
- **Comm:** **all-to-all**, twice per MoE layer (dispatch + combine). All-to-all is the most
  network-hostile collective there is and it's sensitive to routing imbalance — one hot expert
  stalls everyone.
- Brings capacity factor, token-dropping, and auxiliary load-balancing loss into scope.

---

## 3. How they compose — 3D/4D parallelism

The standard frontier recipe, and the thing you should be able to draw:

```
TP  = 8      → inside a node, over NVLink        (highest comm, blocking)
PP  = k      → across nodes                       (lowest comm volume)
DP/FSDP      → across the remaining replicas      (once-per-step, overlappable)
(+ SP/CP for long context, EP if MoE)

total GPUs = TP × PP × DP
```

**The ordering principle is the whole insight:** map each parallelism onto the interconnect
tier that matches its communication intensity. Fastest link gets the chattiest cut.

**Worked example — 400B dense, 512 H100s, 32K context:**
- Model state: 400B × 16 = **6.4 TB**. Over 512×80 GB = 41 TB total, so ~16% on state —
  fits, but activations at 32K context are the real pressure.
- TP=8 (one node) → 50 GB/rank of state, and activations cut 8x with sequence parallel.
- PP=8 across nodes → state per rank down to ~6 GB; need m ≥ 64 microbatches to keep the
  bubble under ~10%.
- DP = 512/(8×8) = 8 replicas, run as FSDP/HSDP for the remaining optimizer-state relief.
- Context parallel on top if 32K activations still don't fit.
- Then check **MFU**: `C ≈ 6ND` for training FLOPs; achieved/peak should land **35–50%**.
  Below 30%, something is wrong and the interviewer wants to hear you say "profile first —
  is it the PP bubble, an unoverlapped all-gather, or a bad kernel?"

---

## 4. What senior interviews actually push on

- **"Your MFU dropped from 45% to 28% overnight, same config. Debug it."** (Stragglers? A
  degraded NIC? ECC errors throttling one GPU? Checkpoint I/O blocking? Data loader stall?
  A single slow rank poisons every collective — collectives run at the speed of the slowest
  participant.)
- **"Why not just use FSDP everywhere?"** (Comm scales with shard count; you fall off the
  bandwidth cliff across nodes. And FSDP alone doesn't solve activation memory at long context.)
- **"You have 30% bubble. Three ways to reduce it."** (More microbatches, interleaved 1F1B,
  fewer PP stages and more TP/FSDP, better stage balancing.)
- **"Global batch size is 4M tokens and PP wants more microbatches. What breaks?"**
  (Large-batch optimization: LR scaling, warmup, and the fact that past a critical batch size
  you're burning compute for no convergence gain.)
- **"A node dies at step 40,000 of 100,000. Walk me through the next 20 minutes."**
  (Checkpoint cadence vs restart cost, async/distributed checkpointing, elastic training,
  whether your data loader is resumable to the exact sample — most aren't, and that's a
  reproducibility bug people underestimate.)

---

## Self-check (answer before tonight's drop)

1. Derive the 16 bytes/param figure from scratch. Now redo it for 8-bit Adam.
2. Why is TP capped at ~8 in practice? Give the answer in terms of collective type and
   frequency, not "it's the convention."
3. You have PP=8 and a 30% bubble. What's your microbatch count, and what's the cheapest fix?
4. FSDP moves 3N bytes vs DDP's 2N, yet FSDP is often *faster* in wall-clock on large models.
   How?
5. Rank these by communication volume per step: TP, PP, DDP, ZeRO-3. Then rank them by
   *sensitivity to interconnect latency*. The orderings differ — explain why.

---

**Tonight:** FDE → evals and measurement. Pretraining → mixed precision, loss spikes, and
numerical stability (the "your loss went to NaN at 3am" genre).
