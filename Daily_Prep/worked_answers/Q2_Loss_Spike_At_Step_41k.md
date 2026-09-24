# Worked Answer — Q2: Loss spike at step 41,000, $400K to restart

> **Track:** Pretraining · ⭐⭐⭐⭐ · Graded at the **Strong Hire (4)** bar
> **Read this as:** the reasoning an interviewer is listening for, with the tells marked.

---

## The question

70B dense, 512 H100s, step 41,000 of 120,000.

- Loss was clean, then rose **2.1 → 3.4 over ~300 steps**
- Partially recovered to **2.6** and **plateaued there** — did not return to trend
- Gradient norm spiked at the same time
- **MFU steady at 42% throughout, including during the spike**
- Full restart ≈ $400K. Checkpoint exists at step 40,000.

*"What do you do, and in what order?"*

---

## Step 1 — Read the three clues before proposing anything

**Clue 1: MFU never moved.**

This is the most informative fact in the problem, and it is there to be used. A steady 42%
means every rank was computing at full speed the entire time. That **rules out the entire
hardware branch**: no straggler, no degraded NIC, no thermal throttling, no checkpoint I/O
contention, no dead rank, no comm-overlap regression. Any of those shows up as an MFU drop
first.

So this is a **numerical, optimization, or data** event. You have eliminated half the search
space from one plot.

> **Signal:** saying explicitly what the steady MFU rules out. Candidates who miss this go on
> to propose hardware checks and burn the interviewer's patience.

**Clue 2: 300 steps, not one step.**

A single corrupted batch produces a one-step spike. A 300-step ramp is a *dynamic developing* —
something growing until it broke. That points at attention-logit growth, a compounding
optimizer instability, or a **contiguous region of bad data** (data is usually contiguous
within a shard, so a corrupted shard spans many consecutive batches).

**Clue 3: partial recovery, then plateau.**

This is the part that decides your response. Full recovery to trend → transient, leave it
alone. **Plateau above trend means the model landed somewhere worse and did not climb out.**
Optimizer state may be damaged — `v` inflated by the spike will suppress effective learning
rate for thousands of steps afterward.

**So the usual advice "many spikes self-heal, don't intervene" does not apply here.** You
already have the evidence that it didn't. Saying that out loud — knowing the default rule
*and* why this case is the exception — is a strong-hire move.

---

## Step 2 — Kill the $400K framing immediately

The $400K number is bait. A full restart is not on the table and shouldn't be discussed.

```
Checkpoint at 40,000; failure at 41,000  →  1,000 steps at risk
1,000 / 120,000 × $400K  ≈  $3.3K of compute
```

**The exposure is roughly $3.3K, not $400K.** Which means the diagnosis budget is effectively
unlimited relative to the stakes — the checkpoint is not going anywhere, and an hour of
investigation costs nothing compared to relaunching blind and hitting the same wall at 41,000
again.

> **Signal:** reframing the cost correctly. An interviewer who quotes $400K is testing whether
> you panic. The right response is arithmetic.

---

## Step 3 — Diagnose before relaunching

Everything here is offline, against logs and the checkpoint.

1. **Per-layer gradient norms across 40,000–41,500.** Is the explosion localized to a few
   layers or global? Localized → architecture or init. Global → LR, data, or numerics.
2. **Attention logit max over time.** If `q·k` magnitudes were climbing across training and
   crossed a threshold around 41,000, that's the single most common cause of spikes in large
   runs, and it explains the 300-step ramp precisely.
3. **The data indices for steps 40,000–41,500.** Pull the actual samples and read them. A
   corrupted shard, an encoding failure, a single pathological document repeated, a language
   switch. *If you cannot map step numbers back to exact samples, that is itself the finding* —
   and it's the highest-priority fix for the rest of the run.
4. **Optimizer state norms.** Did `v` inflate? That tells you whether resuming from 40,000 is
   clean or whether the state needs attention.
5. **Loss on a fixed held-out batch, before and after.** Distinguishes "the model got worse" from
   "these particular batches were hard."

---

## Step 4 — Bisect data vs. optimization

The decisive experiment, and the one that separates people who have done this from people who
have read about it:

> **Resume from step 40,000 and replay the identical batches.**
>
> - Spike **reproduces** → it's the data. You now know exactly which samples.
> - Spike **does not reproduce** → it's stochastic/optimization. Same data, different outcome
>   means the instability was in the dynamics, not the input.

This costs ~1,000 steps of compute — about $3.3K — and it converts a guess into a fact. It is
also the reason §3's data-index logging matters: without it, this experiment is impossible.

---

## Step 5 — Intervene, cheapest first

| If the cause is | Do this |
|---|---|
| **A bad data shard** | Fix or skip the shard, resume from 40,000. Cheapest possible outcome. |
| **Optimization instability** | Resume from 40,000 with reduced LR through the danger zone, tighter gradient clipping, and re-verify the clip is applied after all-reduce and after unscaling (C14). Optionally reset or damp the optimizer's second moment if it's inflated. |
| **Attention logit growth** | The real fix is QK-LayerNorm, and adding it mid-run is not free — it changes the function and needs a short adaptation period. **Name it as the expensive option and say why.** For a run at 34% completion it may still be right; z-loss is the cheaper partial mitigation. |
| **Nothing conclusive** | Resume from 40,000 with a lower LR and tighter clipping, and instrument heavily. Sometimes you resume with a hypothesis rather than an answer — say so honestly rather than inventing a root cause. |

---

## Step 6 — What you change for the rest of the run

The interviewer is usually waiting for this, and most candidates stop at step 5.

- **Alert on gradient norm and attention logit max**, not just loss. Loss is a lagging
  indicator; by the time it moves, the cause is thousands of steps old.
- **Log data indices per step** if you weren't. Non-negotiable.
- **Checkpoint more frequently** through the remaining run — cheap insurance now that you know
  this failure mode exists in this run.
- **Write it down.** A short postmortem in the run log, because you are 79,000 steps from done
  and this will very likely happen again.

---

## What separates the grades

| Grade | What it looks like |
|---|---|
| **1 — No hire** | "Restart from the checkpoint with a lower learning rate." No diagnosis. Or worse, entertains the full restart. |
| **2 — Mixed** | Lists plausible causes but doesn't use the MFU clue, doesn't propose the replay experiment, treats $400K as the real stake. |
| **3 — Hire** | Uses the MFU clue to rule out hardware, diagnoses before relaunching, proposes a sensible intervention ladder. |
| **4 — Strong hire** | All of that, *plus*: reframes the cost as ~$3.3K; reads the plateau as evidence against waiting it out and says why that overrides the usual "spikes self-heal" rule; proposes the replay-identical-batches bisect; names QK-norm as expensive mid-run rather than reaching for it; and closes with what changes for the remaining 79,000 steps. |

---

## The transferable pattern

1. **Mine the given data for what it eliminates**, not just what it suggests. Steady MFU was
   half the answer.
2. **Re-derive the real stakes.** Quoted numbers are often framing, not fact.
3. **Design the decisive experiment** — replaying identical batches cleanly separates two
   hypotheses that otherwise stay entangled forever.
4. **Order interventions by cost and reversibility.**
5. **End with prevention.** The run isn't over.
