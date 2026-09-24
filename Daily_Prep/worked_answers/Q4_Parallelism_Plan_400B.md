# Worked Answer — Q4: "400B dense, 512 H100s, 32K context. Parallelism plan. Defend every number."

> **Track:** Pretraining · ⭐⭐⭐⭐ · Graded at the **Strong Hire (4)** bar
> The canonical pretraining design question. This is the derivation, not the recipe.

---

## Step 0 — State your assumptions out loud

You cannot do this arithmetic without a few numbers the interviewer didn't give you. **Say
them, don't silently assume them** — a stated assumption is a collaboration, an unstated one
is a trap.

```
400B dense, BF16 + FP32 master weights, AdamW
~120 layers, hidden ~16,384, GQA
H100 SXM 80GB, 8 per node → 64 nodes
NVLink within node, InfiniBand between
Activation checkpointing available
```

---

## Step 1 — Memory budget first; it eliminates most designs

```
Model state (C02):  400e9 × 16 bytes  =  6.4 TB
Total cluster HBM:  512 × 80 GB       =  41 TB
```

So model state is ~16% of HBM. Comfortable — which immediately tells you **this is not a
model-state problem, it's an activation problem**, because 32K context is the number they put
in the question and it isn't there by accident.

Naively sharding state evenly across 512 GPUs gives 12.5 GB/rank, leaving ~67 GB for
activations. But you can't shard everything across everything: TP and PP shard parameters,
DP does not. So the real per-rank number depends on the layout, which is what we're choosing.

---

## Step 2 — Place each parallelism on the interconnect that can afford it

The ordering principle, and the thing the question is really testing:

**TP = 8.** Tensor parallel issues two blocking all-reduces per layer per microbatch, sized by
activations. That's the chattiest, least-overlappable traffic you have, so it gets NVLink and
it stops at the node boundary. "Why not TP=16?" — because it would cross to InfiniBand, and
a blocking per-layer collective over IB destroys MFU. Not convention: bandwidth.

**PP = 8, across nodes.** Pipeline parallel sends point-to-point activations at stage
boundaries only — the lowest-volume pattern available — so it's what you put on the slower
link.

**DP = 512 / (8 × 8) = 8 replicas**, running ZeRO-1 (sharded optimizer states) across them.

**Sequence/context parallel on top of TP**, to cut the 32K activation footprint.

---

## Step 3 — Now check it actually fits

```
Params + grads (BF16, 4 bytes), sharded by TP×PP = 64:
    400e9 × 4 / 64                     =  25.0 GB

Optimizer state (FP32 master + m + v = 12 bytes),
sharded by TP×PP×DP = 512:
    400e9 × 12 / 512                   =   9.4 GB
                                          ───────
    Persistent per rank                 ≈  34.4 GB
    Remaining for activations, buffers  ≈  45 GB
```

**Note what ZeRO-1 bought:** without sharding optimizer states across DP, per-rank state is
`400e9 × 16 / 64 = 100 GB` — it does not fit on an 80 GB card at all. That single line is the
justification for the DP-sharding choice, and stating it is what makes the design a derivation
rather than a guess.

**Activations at 32K** are the part to be honest about: they scale with
`microbatch × seq_len × hidden × layers`, plus attention terms. With activation checkpointing,
sequence parallelism dividing by TP=8, and FlashAttention removing the O(n²) score matrix
(C10), ~45 GB is a workable envelope — **but this is the number I'd measure on a single node
before committing, not compute from a formula.** Saying that is a strength, not a hedge.

---

## Step 4 — The pipeline bubble sets your batch size

```
bubble = (p − 1)/(m + p − 1),  p = 8

m = 16  →  7/23  =  30%   ✗
m = 32  →  7/39  =  18%   ✗
m = 64  →  7/71  =  10%   ~
m = 128 →  7/135 =   5%   ✓
```

Take **m = 64** as the working point, with interleaved 1F1B scheduling to push the effective
bubble lower without doubling the microbatch count.

That fixes the global batch:

```
64 microbatches × 8 DP replicas × 1 seq × 32K tokens  ≈  16.8M tokens/step
```

**Then sanity-check it against the critical batch size (C06).** ~16M tokens is large but not
unreasonable for a 400B model. If it were above critical batch size, you'd be burning compute
for no convergence benefit, and the fix would be fewer PP stages plus more TP or FSDP — not
more microbatches. **Making that check unprompted is a strong-hire marker**, because it's where
the pipeline design and the optimization design collide and most candidates treat them as
independent.

---

## Step 5 — Performance target and how you'd verify

```
Target MFU: 35–45%
Effective:  512 × 500 TFLOPS × 0.40  ≈  102 PFLOP/s
Throughput: 102e15 / (6 × 400e9)     ≈  42,500 tokens/s
15T tokens → 15e12 / 42,500          ≈  3.5e8 s  ≈  ~4,000 GPU-days-equivalent wall clock
                                        ≈ 4 months
```

If MFU comes in below 30%, the ordered checklist from C12: one rank or all (straggler vs
systemic) → periodic notch matching the checkpoint interval (I/O blocking) → data loader
starvation → unoverlapped FSDP all-gather → workload change.

**And say what you'd do before the real run:** scale-test at 1/8 size on 64 GPUs to validate
the layout and measure real activation memory, muP-transfer the learning rate from a small
proxy sweep (C14), and set checkpoint cadence from Young/Daly against measured MTBF (C12).
You do not discover your parallelism config is wrong at step 1 of a four-month run.

---

## Step 6 — The follow-ups, pre-empted

| Question | Answer |
|---|---|
| "Why not TP=16?" | Crosses the node boundary; blocking per-layer all-reduces over IB collapse MFU. |
| "Why not pure FSDP?" | Comm scales with shard count; across 512 ranks you fall off the bandwidth cliff, and FSDP alone doesn't solve 32K activation memory. |
| "Reduce the bubble further?" | More microbatches (watch critical batch size), interleaved 1F1B, fewer PP stages with more TP/FSDP, better stage balancing — the last stage carries the LM head and the first carries embeddings, so equal-layer splits are wrong. |
| "MoE instead?" | Changes everything: expert parallelism, all-to-all, and the scaling laws restate in terms of *active* parameters (C06/C08). Different design conversation. |
| "Node dies at step 40k?" | C12: detect via NCCL timeout and heartbeat (a dead rank hangs, it doesn't crash), auto-restart from checkpoint, hot spares, quarantine the node. |

---

## What separates the grades

| Grade | What it looks like |
|---|---|
| **1** | Names strategies without numbers. "We'd use 3D parallelism." |
| **2** | Gets TP=8 and mentions PP, but doesn't check that the layout fits in 80 GB, or picks microbatch count without bubble math. |
| **3** | Full layout with memory math and bubble math, correct interconnect reasoning. |
| **4** | All of that, *plus*: states assumptions up front; shows that ZeRO-1 across DP is what makes it fit at all (100 GB → 34 GB); flags activation memory as measure-don't-derive; checks global batch against critical batch size unprompted; and ends with the pre-run validation plan. |

---

## The transferable pattern

1. **Memory budget first** — it eliminates most designs before you've drawn anything.
2. **Match each parallelism to an interconnect tier** by its communication intensity.
3. **Verify the layout fits**, and show which choice made it fit.
4. **Follow the constraint chain**: bubble → microbatches → global batch → critical batch size.
5. **Say what you'd measure rather than derive**, and what you'd validate before committing
   four months of compute.
