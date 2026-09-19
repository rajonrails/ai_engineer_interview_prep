# C06 — Scaling Laws and Compute Budgeting

> **Day 2, Morning** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** "here's your compute budget, what do you train?" is
> the canonical pretraining design question. It's also the one where candidates recite
> "Chinchilla says 20 tokens per parameter" and then fall apart on the follow-up, which is
> always some version of *"why does nobody actually do that anymore?"*

---

## The One Idea

**Scaling laws tell you how to spend a training budget. They say nothing about your actual
objective, which is almost always total cost of ownership — training plus every inference you
will ever serve.** Chinchilla-optimal is the right answer to a question most teams aren't
asking.

---

## 1. The formulas

**Training compute:**

```
C ≈ 6 · N · D          N = parameters, D = training tokens
```

The 6 = 2 (forward) + 4 (backward). Same 2-FLOPs-per-param as the C01 inference math, with
backward costing twice the forward.

**The loss surface:**

```
L(N, D) = E + A/N^α + B/D^β
```

`E` is irreducible entropy. The two power-law terms are the parameter-limited and data-limited
deficits. **Both terms matter — that was Chinchilla's correction to Kaplan.** Kaplan's original
work held data roughly fixed and concluded "make the model bigger." Chinchilla varied both and
found parameters and data should scale **roughly in equal proportion**: for a fixed C, optimal
N ∝ C^0.5 and D ∝ C^0.5.

The famous heuristic: **D ≈ 20N**.

---

## 2. Worked example — the version to do on a whiteboard

*"You have 10²⁴ FLOPs. What do you train?"*

Chinchilla-optimal:

```
C = 6ND  and  D = 20N
→ C = 120N²
→ N = √(C/120) = √(10²⁴/120) = √(8.33e21) ≈ 9.1e10  ≈ 91B params
→ D = 20N ≈ 1.8T tokens

Check: 6 × 9.1e10 × 1.8e12 ≈ 9.8e23 ✓
```

Now convert to something a budget owner understands:

```
H100 at ~500 TFLOPS peak BF16 dense, 40% MFU → 2e14 FLOP/s effective
10²⁴ / 2e14 = 5e9 GPU-seconds ≈ 58,000 GPU-days
On 1,024 GPUs → ~57 days wall-clock
```

**Being able to walk compute → parameters → tokens → GPU-days → calendar time → dollars is the
whole skill.** The number isn't the point; the chain is.

---

## 3. Why nobody trains Chinchilla-optimal anymore

This is the follow-up, and it's where the interview is actually decided.

**Chinchilla minimizes loss for a fixed *training* budget. It ignores inference entirely.**

If you'll serve the model, inference cost scales with `2ND_inference` — forever. A smaller
model is permanently cheaper to serve. So the real optimization is:

```
total cost = 6·N·D_train  +  2·N·D_inference·(expected lifetime tokens)
```

At any meaningful serving volume, the second term dominates, and the optimum shifts hard
toward **smaller models trained on far more data than Chinchilla-optimal**.

Llama-3-8B was trained on ~15T tokens: **~1,875 tokens per parameter**, roughly 94x past the
Chinchilla ratio. That is not a mistake — it's deliberately accepting worse loss-per-training-FLOP
to buy permanently cheaper inference. The models are "over-trained" only against a metric
nobody with a serving fleet cares about.

**So the correct answer to "what would you train with 10²⁴ FLOPs" is a question:**
*"what's the inference volume, and what latency and memory budget does it have to fit?"*
A candidate who gives 91B/1.8T without asking that has recited a paper. A candidate who asks
it is doing the job.

---

## 4. What breaks the scaling laws

| Factor | Effect |
|---|---|
| **Data repetition** | The laws assume unique tokens. Repeated epochs give diminishing returns — roughly, up to ~4 epochs is close to fresh data, past that the value decays toward nothing. Matters enormously now that high-quality web text is a binding constraint. |
| **Data quality** | Filtering and dedup shift the whole curve. A better corpus beats a bigger model at the same compute, which is why data work is the highest-leverage pretraining work. |
| **MoE** | Decouples parameter count from FLOPs/token. The laws need restating in terms of *active* parameters, with total params as a separate axis. |
| **Architecture changes** | The constants (A, B, E) move; the exponents are fairly robust. Useful, because it means you can compare architectures at small scale and extrapolate. |
| **Distillation** | A student trained on teacher logits beats its own from-scratch scaling curve. Breaks the framework entirely. |
| **Post-training** | Scaling laws describe pretraining loss. Downstream capability after SFT/RL is a different curve, and loss is an imperfect proxy for it. |

---

## 5. The things that ride along

- **Critical batch size.** Past a certain batch size, doubling it stops halving the step count —
  you're burning compute for nothing. It grows over training (larger batches are efficient
  later), which is why batch-size ramps exist. If asked "why not batch size 100M," this is why.
- **muP.** Parametrize so the optimal LR is width-invariant; tune on a small proxy and transfer.
  This is the answer to *"you can only train the big model once, how do you pick hyperparameters?"*
- **Scaling-law fits as a planning tool.** Train a ladder of small models (say 50M → 1B), fit
  `L(N,D)`, extrapolate to the target. Standard practice. Be honest about the limits:
  extrapolating more than ~1-2 orders of magnitude is uncertain, and emergent downstream
  behavior isn't predicted by loss at all.
- **Compute-optimal ≠ best model you can build.** It's the best model *per training FLOP*,
  which is a constraint, not a goal.

---

## 6. Interview framing

- *"You have 10²⁴ FLOPs, what do you train?"* → do the arithmetic, then immediately ask about
  inference volume and serving constraints. Present both answers.
- *"Why did Llama-3 train 8B on 15T tokens when Chinchilla says 160B tokens?"* → inference-aware
  optimization; total cost of ownership; permanently cheaper serving.
- *"You have 10x the compute. Bigger model or more data?"* → roughly equal split under
  Chinchilla (√10 ≈ 3.2x each), but skew toward data if you serve at volume, and check whether
  you *have* 3.2x more unique high-quality tokens — increasingly you don't, which pushes toward
  more epochs, better filtering, or synthetic data.
- *"How do you pick the LR for a model you can only train once?"* → muP transfer from a proxy,
  plus a small LR sweep at reduced scale, plus a conservative warmup.

---

## Self-check

1. Derive N and D for a 3e23 FLOP budget, then convert to GPU-days on 512 H100s at 40% MFU.
2. Why is Chinchilla-optimal the wrong target for a model you'll serve to millions of users?
   Write the total-cost expression.
3. You have 10x compute but only 1.5x more unique tokens. What do you do?
4. What does critical batch size mean and why does it grow during training?
5. Name three things that break the standard scaling-law framework.
