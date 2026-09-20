# C09 — Fine-Tuning: When, When Not, and What It Actually Costs

> **Day 3, Morning** · Track: FDE · ⭐⭐⭐⭐
> **Read time:** ~20 min. **Why now:** "we want to fine-tune" is the single most common
> customer request an FDE receives, and the single most common request an FDE should talk
> them out of. How you handle it is a direct test of whether you're an order-taker or an
> engineer.

---

## The One Idea

**Fine-tuning changes *behavior*, not *knowledge*. Most customers asking for it actually want
knowledge, and the right answer is retrieval.**

The second idea, equally important in the field: **fine-tuning is a commitment, not an
experiment.** You don't ship a fine-tune, you adopt a model — with versioning, an eval suite,
a retraining path, and a migration plan for when the base model is deprecated. Customers
consistently price the training run and not the ownership.

---

## 1. The ladder — always climb in this order

| Rung | Cost to try | Use when |
|---|---|---|
| **1. Better prompt** | minutes | Always first. Most "the model can't do this" is an under-specified prompt. |
| **2. Few-shot examples** | minutes | Format and style are inconsistent. 3-5 examples fix an enormous fraction of complaints. |
| **3. Retrieval** | days | The model lacks *information*. This is the answer to almost every "it doesn't know our product" complaint. |
| **4. Prompt/context engineering + tool use** | days | The task needs structure, decomposition, or external actions |
| **5. Fine-tune** | weeks + ongoing | Behavior is still wrong after 1-4, *and* you have data, *and* you can commit to owning it |

**The interview answer is not "never fine-tune."** It's "fine-tuning is rung 5, and I want to
know why rungs 1-4 failed before I spend your quarter on it." Blanket refusal reads as
dogmatic; a clear ladder reads as experienced.

---

## 2. What fine-tuning is genuinely good at

- **Format and structure adherence.** Reliable JSON/XML conformance to a specific schema,
  consistently, without prompt gymnastics.
- **Tone and style.** Matching a brand voice, a clinical register, a legal register.
- **Narrow task specialization.** One classification task, done extremely well and cheaply.
- **Cost and latency reduction — the underrated one.** Distill a large model's behavior on
  *your* task into a small one. A fine-tuned small model matching a large model's quality on
  a narrow task is often a 10-20x cost reduction. Revisit C01: this is the lever that actually
  moves a serving bill.
- **Tool-calling reliability.** Getting the right tool with the right args, consistently.
- **Domain jargon and conventions** that no amount of prompting makes natural.

## 3. What it is bad at — and the failure everyone hits

- **Adding facts.** Fine-tuning on documents teaches the *shape* of the documents, not their
  contents, and what it does memorize it memorizes unreliably and unverifiably. The model
  becomes *more* confident and *no* more correct — arguably the worst possible outcome.
  **This is the misconception to name explicitly.**
- **Anything that changes.** Prices, policies, inventory, staff. Retrain-on-every-change is not
  an architecture.
- **Attribution.** RAG can cite. A fine-tune cannot tell you where it got something.
- **Access control.** You cannot un-train one tenant's data out of a shared model. If different
  users may see different data, retrieval with filters is the only correct answer, and this is
  a *security* argument, not a preference.

---

## 4. Mechanics worth knowing precisely

**LoRA.** Freeze `W`, learn a low-rank update `ΔW = BA` where `B` is `d_out × r` and `A` is
`r × d_in`. Trainable parameters per matrix:

```
r × (d_in + d_out)     vs     d_in × d_out for full fine-tuning
```

For `d=4096, r=16`: `16 × 8192 = 131K` vs `16.7M` — about **0.8%** of the parameters. Typical
`r` is 8-64; higher rank for tasks that need more behavioral change, lower for style. Adapters
are small enough to swap per-customer at serving time, which is a real architectural advantage:
one base model, many tenants.

**QLoRA.** 4-bit quantized frozen base + LoRA adapters in higher precision. Makes single-GPU
fine-tuning of large models feasible.

**Data.** Quality dominates quantity. ~1,000 excellent examples usually beat 50,000 scraped
ones. For style/format tasks a few hundred can suffice. The expensive part is curation, not
compute — and that's the part customers underestimate.

**Catastrophic forgetting.** Narrow fine-tuning degrades general capability. **Always eval on
general benchmarks too, not just your task** — otherwise you ship a model that aces your
classification and has quietly lost its ability to handle anything off-script. Mitigations:
lower LR, fewer epochs, LoRA over full FT, and mixing general data into the training set.

---

## 5. The conversation to have with a customer

When someone says "we want to fine-tune," the questions that matter:

1. **What's failing right now, specifically?** Show me ten failures. (Often they're retrieval
   failures, or prompt failures, or the eval set is wrong.)
2. **Knowledge or behavior?** If they can't answer, ask: "if the correct answer were pasted
   into the prompt, would the model get it right?" **Yes → retrieval. No → maybe fine-tuning.**
   That single question resolves most cases.
3. **What data do you have?** Not "we have 100K support tickets" — how many are *correct,
   consistent examples of the behavior you want*? Usually far fewer than they think.
4. **How will you know it worked?** No eval set → no fine-tune. This is non-negotiable and
   saying so is a strong-hire signal.
5. **Who owns it in six months?** When the base model is deprecated, who retrains?

**Then propose the cheap experiment first**: better prompt plus retrieval, measured on their
eval set, in one week. If that closes the gap, you saved them a quarter. If it doesn't, you now
have a baseline, an eval set, and a genuine case for fine-tuning — and you've earned the
credibility to run it.

---

## Self-check

1. Customer: "fine-tune on our 50K support tickets so it knows our product." Response?
2. Give the one-sentence test for knowledge vs. behavior.
3. LoRA trainable params for `d_in=d_out=4096, r=32`. What fraction of full FT?
4. Why is multi-tenant data a *security* argument against fine-tuning?
5. Name the case where fine-tuning is clearly correct and RAG cannot substitute.
6. What do you eval after fine-tuning, besides the target task, and why?
