# C16 — Post-Training and the Pretraining Handoff

> **Day 4, Evening** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** pretraining and post-training are separate teams, and
> the interview tests whether you understand the interface. "What does post-training need from
> you?" is a staff-level question, and the boundary has been getting blurrier every year.

---

## The One Idea

**Pretraining decides what the model *can* do; post-training decides what it *does*.** You
cannot post-train in a capability that isn't latent in the base model — but you can very
easily post-train *out* capability that is.

So the pretraining engineer's job doesn't end at the final checkpoint. It ends when the base
model is demonstrably **post-trainable**: broadly capable, well-calibrated, and not
over-narrowed by aggressive filtering.

---

## 1. The pipeline

```
Pretraining → [base model] → SFT → preference optimization → [+ RLVR] → eval/safety → ship
```

**SFT (supervised fine-tuning).** Train on (prompt, good response) pairs to convert a text
completer into something that follows instructions. Findings that hold up consistently:
**data quality overwhelmingly dominates quantity** — a few thousand excellent, diverse,
carefully-curated examples routinely beat hundreds of thousands of scraped ones. Diversity of
*task type* matters more than volume within a type.

**Preference optimization.** SFT teaches one good answer. Preferences teach *better vs worse*,
which is a much richer signal for things like tone, helpfulness, refusal calibration, and
formatting — qualities that are easier to compare than to specify.

---

## 2. RLHF vs DPO — know both mechanisms

**RLHF (classic, PPO-based), three stages:**

1. Collect preference pairs: for a prompt, humans rank response A vs B.
2. Train a **reward model** `r_θ(x,y)` on those comparisons (Bradley-Terry).
3. Optimize the policy with PPO against that reward, with a KL penalty to the reference:

```
objective = E[ r_θ(x,y) ] − β · KL( π_policy ‖ π_ref )
```

**The KL term is the part to understand, not just cite.** Without it the policy drifts to
exploit the reward model — the classic **reward hacking** failure, where responses score
brilliantly and are gibberish, or where the model discovers that longer answers score higher
and inflates everything. β controls how far you'll let it wander from the SFT model.

**DPO (Direct Preference Optimization).** Skips the reward model entirely. The insight: the
optimal policy under the RLHF objective has a closed form in terms of the reward, so you can
invert it and express the reward implicitly through the policy itself — then optimize
directly on preference pairs:

```
L = −log σ( β·[ log π(y_w|x)/π_ref(y_w|x) − log π(y_l|x)/π_ref(y_l|x) ] )
```

Far simpler: no reward model, no sampling loop, stable supervised-style training. Trade-offs
to name: no reward model means no way to score *new* samples, it's off-policy (fixed
preference data rather than fresh generations), and it can over-optimize toward the preferred
responses' surface features. Variants — IPO, KTO (works from binary good/bad rather than
pairs), ORPO (folds SFT and preference into one stage), SimPO — each attack one of those.

**RLVR (RL with verifiable rewards).** Where correctness is *checkable* — math with a known
answer, code that must pass tests, formal proofs — you skip human preference entirely and use
the verifier as reward. Much cleaner signal, no reward model to hack, and it is where a large
share of recent reasoning gains has come from. **The limit is the obvious one: it only applies
where a verifier exists**, and building good verifiers is itself the hard part.

---

## 3. What post-training needs from pretraining

This is the actual interface question, and the part most candidates haven't thought about.

| Requirement | Why | What pretraining controls |
|---|---|---|
| **Latent capability** | You cannot SFT in reasoning that isn't there | Data mixture — code and math in pretraining improve post-trained reasoning broadly (C08) |
| **Calibration** | A base model that is confidently wrong is hard to make honest | Over-filtering and heavy repetition damage calibration |
| **Format diversity** | The base should have seen dialogue, lists, structured data, code | Corpus composition |
| **Not over-narrowed** | Aggressive quality filtering can strip registers and domains post-training then can't recover | The filtering-vs-diversity trade-off from C08 |
| **A stable, documented checkpoint** | Post-training needs reproducibility and the exact tokenizer | Checkpoint contents (C12) |
| **Headroom** | Annealed and midtrained models behave differently under SFT | Annealing recipe (C08) |

**The boundary is genuinely blurring.** Midtraining/annealing already injects
instruction-like and high-quality data late in pretraining, and long-context extension (C10)
often happens in that phase too. "Where does pretraining end" is a legitimate architectural
question now, and having an opinion on it reads as current.

---

## 4. The costs and failure modes

- **Alignment tax.** Post-training can reduce raw capability on some benchmarks even as it
  improves usefulness. Track base-model evals through the whole pipeline, not just chat evals,
  or you won't see it.
- **Reward hacking.** Verbosity inflation, sycophancy, hedging, formatting tics that score well.
  Detection: humans reviewing high-reward samples, which is unglamorous and necessary.
- **Mode collapse / diversity loss.** Heavily preference-optimized models produce
  lower-variance output. Costs you on creative tasks and on anything using the model to
  generate training data (see model collapse, C08).
- **Catastrophic forgetting.** Same mechanism as C09, at a larger scale.
- **Eval mismatch.** Base models are evaluated few-shot on benchmarks; chat models on
  instruction-following and preference. A regression can hide entirely in the gap between the
  two suites.

---

## 5. Interview framing

- **"What does post-training need from your base model?"** → the table in §3. Lead with latent
  capability and calibration.
- **"RLHF or DPO?"** → DPO for simplicity, stability, and when you have fixed preference data;
  RLHF/PPO when you need an explicit reward model to score fresh samples or run online; RLVR
  wherever a verifier exists. Don't pick one dogmatically — name the deciding factor.
- **"Your post-trained model is worse at math than the base."** → alignment tax and/or SFT
  mixture lacking math; check base evals through the pipeline, mix capability-preserving data
  into SFT.
- **"The model got verbose and sycophantic after RLHF."** → reward hacking; check whether the
  reward model rewards length, tighten the KL penalty, add length-controlled preference data.
- **"Where does pretraining end and post-training begin?"** → honest answer: it's a continuum
  now, and annealing is the seam.

---

## Self-check

1. Why does the KL penalty exist in the RLHF objective? What happens at β=0?
2. Write DPO's core idea in two sentences. What does it give up versus a reward model?
3. When is RLVR the right choice, and what's its hard limit?
4. Name four things post-training needs from a base model.
5. Post-trained model regresses on math. Two hypotheses and how you'd distinguish them.
6. Why can aggressive pretraining quality filtering hurt *post-training* specifically?
