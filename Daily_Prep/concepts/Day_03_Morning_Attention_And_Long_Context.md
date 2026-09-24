# C10 — Attention Variants, FlashAttention, and Long Context

> **Day 3, Morning** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** this is the "do you actually know the architecture"
> round. Expect to be asked to derive shapes on a whiteboard, explain FlashAttention without
> hand-waving, and say precisely how a 8K model becomes a 128K model.

---

## The One Idea

**Attention's problems are memory problems, and each variant attacks a different one.**
GQA attacks *KV cache size*. FlashAttention attacks *activation memory and HBM traffic*.
Sliding-window and sparse patterns attack the *O(n²) compute*. RoPE scaling attacks
*positional generalization*. Knowing which problem each solves is the whole topic — candidates
who list them as interchangeable "optimizations" get marked down.

---

## 1. MHA → MQA → GQA

For each head: `Q, K, V ∈ R^(n × d_head)`, `Attn = softmax(QKᵀ/√d_head)V`.

The `√d_head` is not decoration: `q·k` is a sum of `d_head` products, so its variance grows
with `d_head`. Without the scale, logits grow with dimension and softmax saturates into a
near-one-hot distribution with vanishing gradients.

| Variant | KV heads | KV cache | Quality |
|---|---|---|---|
| **MHA** | `n_heads` | baseline | baseline |
| **MQA** | 1 | `1/n_heads` | measurable degradation, and training instability reported |
| **GQA** | `g` groups (typically 8) | `g/n_heads` | ≈ MHA |

GQA is the settled answer: near-MHA quality at a large fraction of MQA's savings. Revisit the
C01 arithmetic — Llama-3-70B at 320 KB/token with GQA versus ~2.6 MB/token without. **Since
KV cache caps batch size and batch size determines serving economics, this is an architecture
decision made at pretraining time that determines your inference cost forever.** That framing —
a pretraining choice with permanent serving consequences — is what a senior answer sounds like.

---

## 2. FlashAttention

**What it is not:** an approximation. Output is numerically equivalent to standard attention.

**The problem:** naive attention materializes the `n × n` score matrix in HBM. At `n = 32K`
that's ~1B entries per head per sequence. The kernel is bound by HBM reads and writes, not by
arithmetic — the GPU is idle waiting on memory.

**The mechanism:** tile Q, K, V into blocks that fit in on-chip SRAM, and compute softmax
**incrementally** using the online-softmax trick — maintain a running max and a running sum,
rescaling accumulated output as new blocks arrive. The full score matrix is never written to
HBM.

```
memory:  O(n²)  →  O(n)
HBM traffic: reduced by roughly the tile factor
FLOPs: unchanged (slightly increased — backward recomputes scores instead of storing them)
```

**The lesson to state out loud:** FlashAttention does *more* arithmetic and is *much* faster,
because the bottleneck was never arithmetic. That is the same roofline reasoning as C01's
decode analysis, and connecting the two shows you understand the principle rather than the
trick. Later versions improve work partitioning and warp scheduling; FlashAttention is why
long-context training is feasible at all.

---

## 3. Positional encoding and RoPE

**RoPE** rotates Q and K by an angle proportional to absolute position. Because a rotation by
`mθ` dotted with a rotation by `nθ` depends only on `m − n`, **relative position falls out of
the dot product** while only absolute position is ever applied. That's the elegance.

Frequencies per dimension pair:

```
θ_i = base^(-2i/d)          base typically 10,000
```

Low `i` → high frequency → local/fine-grained position. High `i` → low frequency → long-range.
That frequency split is exactly what the extension methods below exploit.

**ALiBi** takes a different route: add a linear penalty `−m·|i−j|` to attention scores, with a
per-head slope. Extrapolates to longer sequences naturally, but is effectively a recency bias.

---

## 4. Extending context

The problem: train at 8K, and positions beyond 8K are out of distribution. The model has
genuinely never seen those rotation angles.

| Method | Mechanism | Trade-off |
|---|---|---|
| **Position Interpolation** | Scale positions down by `s` so 32K maps into the trained 8K range | Simple, works with brief fine-tuning. Compresses *all* frequencies, degrading fine-grained local resolution. |
| **NTK-aware scaling** | Increase the RoPE `base` instead | Often works with no fine-tuning at modest ratios |
| **YaRN** | Frequency-dependent: leave high-frequency dims alone, interpolate low-frequency dims, plus an attention-temperature correction | Best quality per unit of fine-tuning. The reasoning — local resolution lives in high frequencies, so don't touch it — is the part worth being able to explain. |
| **Sliding window** | Each token attends to the last `w` tokens | O(n·w). Information propagates across layers, so effective receptive field is `w × n_layers`. |
| **Attention sinks** | Always keep the first few tokens in the window | Models dump excess attention mass on the first tokens; evict them and streaming quality collapses. Cheap fix, surprising failure, good detail to know. |

**The honest caveat interviewers want to hear:** context *length* is not context *capability*.
A model extended to 128K may retrieve a single fact from position 90K (needle-in-haystack)
while failing to reason over the whole window. Evaluate long context with multi-hop and
aggregation tasks, not just needle tests — those are close to saturated and mostly measure
retrieval, not reasoning.

---

## 5. Interview framing

- **"Derive attention's memory cost."** `O(batch × heads × n²)` for scores; explain why Flash
  removes it without approximating.
- **"Why GQA over MQA?"** Near-MHA quality, most of the cache savings, better stability.
- **"How do you take an 8K model to 128K?"** Choose a scaling method (say why YaRN over PI),
  fine-tune briefly on long sequences, watch short-context regression, and evaluate with
  aggregation tasks rather than needle tests.
- **"Why divide by √d_head?"** Variance of the dot product grows with `d_head`; without it
  softmax saturates and gradients vanish.
- **"Why do attention sinks exist?"** Softmax must sum to 1, so a head with nothing to attend
  to has to put its mass somewhere; the first tokens absorb it.

---

## Self-check

1. KV cache per token for a 32-layer, 32-head, 8-KV-head, `d_head=128` model in BF16.
2. Explain FlashAttention's online softmax. Why is it exact, not approximate?
3. Why is FlashAttention faster while doing more FLOPs?
4. What does YaRN do that Position Interpolation doesn't, and why does it matter?
5. Why does evicting the first tokens from a sliding window collapse quality?
6. Your 128K model scores 99% on needle-in-haystack and fails a multi-hop question over
   40K tokens. What's going on?
