# C03 — Evals: Measuring Whether an LLM System Actually Got Better

> **Day 1, Evening** · Track: FDE · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why this matters:** this is the highest-signal FDE topic after
> inference economics, and it's where most candidates are weakest. Anyone can say "we built
> an eval set." Almost nobody can tell you how many examples they needed, or why their
> LLM-judge scores were inflated.

---

## The One Idea

**In a deterministic system, tests verify what you already know. In an LLM system, evals are
how you find out anything at all — they are the measurement instrument, and a bad instrument
is worse than none because it produces confident wrong numbers.**

The failure mode isn't "we didn't test." It's "we tested on 20 cherry-picked examples,
saw 18/20, shipped, and regressed silently for six weeks." Both numbers *looked* fine.

**Interview framing you should internalize:** the person who owns evals owns the roadmap,
because they're the only one who can say whether a change was an improvement. Say that out
loud in an interview and watch the tone of the conversation change.

---

## 1. The sample-size math nobody does

This is the single most differentiating thing you can bring to an eval conversation. Most
candidates report "82% accuracy" on 50 examples without a confidence interval. Ask them if
81% is worse and they can't answer.

95% confidence interval half-width for a proportion, worst case (p ≈ 0.5):

```
half-width ≈ 1.96 × √(p(1-p)/n)  ≈  0.98/√n
```

| n | 95% CI half-width | What you can actually detect |
|---|---|---|
| 20 | **±22 pp** | Essentially nothing. This is vibes with a number attached. |
| 50 | ±14 pp | Only catastrophic regressions |
| 100 | ±10 pp | Large changes |
| 400 | ±5 pp | Meaningful product changes |
| 1,000 | ±3 pp | Fine-grained iteration |

**So: "we improved from 82% to 86% on our 100-example eval set" is not a result.** The CIs
overlap almost entirely. Saying this in an interview is a strong-hire signal.

**The senior move — use a paired design.** Run both variants on the *same* examples and count
only the cases where they disagree (McNemar's test). Per-example difficulty variance cancels
out, so you detect the same effect with roughly an order of magnitude fewer examples. If
you're A/B-ing prompts, you should almost always be doing this rather than comparing two
independent accuracy numbers.

---

## 2. The eval ladder

Use all four. They catch different failures and cost wildly different amounts.

| Level | What it is | Cost | Catches | Misses |
|---|---|---|---|---|
| **Assertions / unit evals** | Deterministic code checks: valid JSON, schema conformance, required field present, no PII, latency budget, tool called with valid args | ~free, runs in CI | Format breakage, contract violations, crashes | Anything about quality |
| **LLM-as-judge** | A model grades output against a rubric or picks between two candidates | ~cents per example | Fluency, relevance, instruction-following, faithfulness | Subtle factual errors; its own biases |
| **Human review** | SMEs label a sample against a rubric | expensive, slow | Ground truth, domain correctness | Doesn't scale; inter-rater drift |
| **Online / production** | A/B test on real traffic: task completion, escalation rate, thumbs, retry rate | slow, real risk | What actually matters | Slow feedback; confounded; needs volume |

**The pipeline that works:** assertions gate every PR → LLM-judge runs on a few hundred
examples nightly → humans label a rotating sample to keep the judge calibrated → online A/B
for anything significant. Each level calibrates the one above it.

---

## 3. LLM-as-judge: the biases you must name

If you propose LLM-as-judge without naming its failure modes, a good interviewer will assume
you've never actually run one.

| Bias | What happens | Mitigation |
|---|---|---|
| **Position bias** | In pairwise comparison, judges systematically favor the first (or second) option — this effect can be large | Run both orderings, average. If the verdict flips, it's a tie. |
| **Verbosity bias** | Longer answers score higher regardless of quality | Control for length in the rubric; penalize padding explicitly |
| **Self-preference** | Models rate their own generations higher | Use a different model family as judge than the one being evaluated |
| **Rubric drift** | Scores shift when you change the prompt, model version, or temperature | Pin the judge model+version. Treat a judge change as a migration, with a re-baseline. |
| **Score compression** | Everything lands on 4/5 and you lose resolution | Prefer pairwise preference over absolute scores; force a rubric with concrete anchors |

**The calibration rule:** measure your judge against human labels on a held-out set and report
its agreement rate. If your judge agrees with humans 70% of the time, every number it produces
carries that error. Most teams never do this, and it's a strong thing to volunteer.

---

## 4. Decompose the pipeline — the #1 RAG eval mistake

"Our RAG accuracy is 71%" is a useless number because it conflates independent failures.
Measure each stage separately:

| Stage | Metric | Diagnostic question |
|---|---|---|
| **Retrieval** | recall@k — is the answer-bearing chunk in the top k at all? | If this is 60%, nothing downstream can save you. **Fix here first.** |
| **Reranking** | precision@k, MRR — is it near the top? | Right doc retrieved but ranked 40th → rerank, don't touch the prompt |
| **Generation (faithfulness)** | is every claim supported by the retrieved context? | Catches hallucination *given good context* |
| **Generation (answer relevance)** | does it actually answer the question asked? | Catches evasive/off-target answers |
| **End-to-end** | task success | The only number the customer cares about |

**Retrieval recall@k is the ceiling on everything else.** If the chunk isn't retrieved, no
prompt engineering, no bigger model, no reranker recovers it. Teams routinely burn weeks on
prompt tuning when their recall@10 is 55%. Checking that number first is exactly the instinct
FDE interviews are screening for.

---

## 5. Where eval sets come from

**Not from your imagination.** Synthetic questions you invent test the system you *imagined*,
not the one users hit.

1. **Mine production traces.** Sample real queries, stratified by intent and by outcome.
   Oversample the failures.
2. **Every bug becomes a test case.** This is the highest-ROI habit in applied AI. An incident
   that doesn't produce an eval row will recur.
3. **Adversarial and edge slices.** Prompt injection attempts, empty retrieval, contradictory
   sources, out-of-scope questions, multi-hop questions, the longest 1% of inputs.
4. **Slice your metrics.** Aggregate accuracy hides everything. Report by query type, by
   document source, by customer, by input length. A 3pp aggregate drop is often one slice
   falling off a cliff.
5. **Freeze a holdout.** If you tune against a set continuously, you've overfit to it. Keep a
   set you look at rarely.

---

## 6. How this shows up in interviews

- **"How do you know your change helped?"** → paired design, CI on the delta, sliced results.
- **"Your eval says 94% but users complain."** → distribution mismatch between eval set and
  real traffic; the metric doesn't capture what users value; or you overfit to the eval set.
- **"How many examples do you need?"** → depends on the effect size you need to detect; quote
  the `0.98/√n` rule; propose paired comparison to cut n.
- **"Customer has no labeled data and wants to launch in 3 weeks."** → this is the real FDE
  question. Start from production traces + SME review of 100 examples; build assertions
  immediately since they're free; get a judge calibrated against those 100; ship with online
  guardrails and a rollback plan. **Do not** propose a 6-month labeling project.

---

## Self-check

1. Your eval moved from 82% → 86% on n=100. Is that a real improvement? Show the arithmetic.
2. Name three LLM-judge biases and the concrete mitigation for each.
3. RAG end-to-end accuracy is 64%. What do you measure next, and in what order — and why is
   the prompt the *last* thing you touch?
4. Why is a paired eval design so much more sensitive than comparing two independent accuracy
   numbers?
5. A customer wants "95% accuracy" in the contract. What do you say?
