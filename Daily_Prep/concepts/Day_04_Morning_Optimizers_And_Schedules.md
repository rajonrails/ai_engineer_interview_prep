# C14 — Optimizers, Learning-Rate Schedules, and muP

> **Day 4, Morning** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** "you can train the big model once — how do you pick
> the hyperparameters?" is the question that most cleanly separates people who have run
> frontier training from people who have fine-tuned. It has a real answer, and most candidates
> don't know it.

---

## The One Idea

**Optimizer and schedule choices are where a single wrong number silently costs you the entire
run.** There is no unit test for "the learning rate was 2x too high" — you find out 30,000
steps in, or worse, you don't, and you just get a mediocre model and never know why.

---

## 1. AdamW, precisely

```
m_t = β₁·m_{t-1} + (1-β₁)·g_t                 first moment
v_t = β₂·v_{t-1} + (1-β₂)·g_t²                second moment
m̂ = m_t/(1-β₁ᵗ),  v̂ = v_t/(1-β₂ᵗ)            bias correction
θ_t = θ_{t-1} - lr·( m̂/(√v̂ + ε) + λ·θ_{t-1} )
```

Three things to be precise about:

**Bias correction.** `m` and `v` start at zero, so early estimates are biased toward zero.
Without correction the first steps take wildly wrong-sized updates. Interacts with warmup —
both are protecting the same early-training window.

**The W in AdamW — decoupled weight decay.** L2 regularization added to the *gradient* gets
divided by `√v̂` along with everything else, so parameters with large gradients get *less*
decay — the opposite of the intent. AdamW applies `λ·θ` directly to the weight update,
decoupled from the adaptive scaling. **"Why AdamW and not Adam with L2" is a standard
question and this is the whole answer.** Also: don't decay biases and LayerNorm gains.

**β₂ = 0.95, not 0.999.** The PyTorch default of 0.999 averages the second moment over
~1000 steps. At large scale that window is too long — the optimizer responds sluggishly to
genuine gradient-scale changes, and a single bad batch stays in `v` for a long time. Large
runs typically use **β₁=0.9, β₂=0.95**, and knowing that this is a deliberate deviation from
the default is a small but real signal.

**ε.** Nominally 1e-8, but at scale gradients can be small enough that ε dominates; values
down to 1e-15 appear in large runs. Its placement (inside vs outside the square root) also
differs between implementations — worth checking rather than assuming.

**Memory.** `m` and `v` in FP32 are 8 of the 16 bytes/param from C02. Halving optimizer state
(8-bit Adam) is one of the few free-ish memory wins.

---

## 2. Gradient clipping

Clip by **global** norm across all parameters, typically to 1.0:

```
if ||g||₂ > c:  g ← g · c/||g||₂
```

Global, not per-parameter — per-parameter clipping distorts the gradient direction.

The detail that catches people: in distributed training, clipping must happen **after**
gradient all-reduce and **after** unscaling (if using FP16 loss scaling, per C04). Clip before
the all-reduce and each rank clips its local gradient, which is not the same operation and
silently changes your effective update. Also: **log the pre-clip gradient norm.** It's the
single most diagnostic scalar in a training run, and clipping frequency is your early warning
that something is going wrong.

---

## 3. Learning-rate schedules

**Warmup, then decay.** Warmup is typically 1-2% of total steps, linear from 0 to peak.

Why warmup exists: early in training `v̂` is estimated from very few samples and is unreliable,
so adaptive scaling is untrustworthy exactly when the model is most fragile. Large early
updates push the model into a bad region it may never leave. Warmup is cheap insurance, and
its absence is a common cause of early loss spikes.

| Schedule | Shape | Notes |
|---|---|---|
| **Cosine decay** | Peak → ~10% of peak following a cosine | Long-time default. **Requires knowing total steps in advance** — that's its real drawback. |
| **WSD (warmup-stable-decay)** | Warmup → long constant LR → short sharp decay | Increasingly preferred: you can stop at *any* point in the stable phase, run the short decay, and get a usable model. Decouples "how long do we train" from the schedule, and pairs naturally with the data annealing from C08 — decay the LR while upweighting high-quality data. |
| **Inverse sqrt** | `lr ∝ 1/√t` | Older, no total-steps requirement |

**Peak LR vs batch size.** For Adam-family optimizers, LR scales roughly with **√batch_size**
(linear scaling is the SGD rule and over-scales for Adam). And recall the critical batch size
from C06 — past it, larger batches buy you nothing regardless of how you scale the LR.

---

## 4. muP — the actual answer to "you only get one run"

**The problem.** Optimal LR shifts as you change model width. So you tune on a 100M proxy,
scale to 70B, and your carefully-tuned LR is now wrong — usually too high. Everyone's
historical workaround was "use a smaller LR for bigger models and hope."

**The fix.** Maximal Update Parametrization re-parametrizes initialization scale, per-layer
learning rates, and output multipliers as functions of width, such that the optimal
hyperparameters become **width-invariant**. Tune on a small proxy, transfer directly.

Roughly: input/embedding layers, hidden layers, and output layers each get different
width-scaling of init variance and LR, chosen so that activation and update magnitudes stay
O(1) as width grows.

**Why it matters in an interview:** it converts hyperparameter selection from an expensive
gamble into a cheap, principled sweep. When asked "how do you pick the LR for a model you can
train once," the strong answer is: **muP-transfer from a proxy sweep, cross-checked with a
short run at intermediate scale, with a conservative warmup as insurance.** The weak answer is
"we'd use 3e-4" — a number that came from a paper about a different-sized model.

---

## 5. What's moving

- **8-bit Adam** — quantized optimizer states, big memory win, quality impact usually small.
- **Shampoo / SOAP / Muon** — approximate second-order methods using preconditioning matrices.
  Meaningfully better steps-to-target in reported results, at the cost of extra compute and
  memory per step and more implementation complexity. Active area; worth being able to say
  what they do (precondition the gradient using curvature structure rather than just
  per-coordinate variance) and that the trade-off is per-step cost versus step count.
- **Schedule-free methods** — remove the schedule entirely via averaging. Attractive because
  they remove the need to commit to a total step count up front.

---

## Self-check

1. Write the AdamW update. Why is decoupled decay different from Adam + L2?
2. Why β₂=0.95 rather than the 0.999 default?
3. Why warmup? Which other mechanism protects the same window?
4. Where in a distributed step must gradient clipping happen, and what breaks otherwise?
5. Batch size 4x. How do you change the LR for Adam, and why not linear?
6. Explain muP in three sentences to someone who has only ever fine-tuned.
7. What does WSD buy you over cosine, and how does it pair with data annealing?
