# C18 — Telling the Pretraining Story: Scope, Credit, and Leveling

> **Day 6, Evening** · Track: Pretraining · ⭐⭐⭐⭐
> **Why this matters for you specifically:** you have 18 months of hands-on pretraining, which
> is a rare and expensive credential. How you narrate it decides your level and your offer,
> and pretraining has a structural problem no other specialty has — **the work is collective,
> the runs are shared, and the headline numbers belong to a team.**

---

## The One Idea

**"We trained a 70B model" conveys almost nothing.** Hundreds of people can say it. The
interview needs to know what *you* specifically owned, decided, and changed — at a granularity
fine enough to be probed and coarse enough to matter.

Two failure modes, asymmetric in cost:

- **Overclaiming** — presenting the team's MFU number or the run's success as yours. Fatal
  when probed, and it *will* be probed. Frontier labs are small; your interviewer may know
  someone on that team.
- **Underclaiming** — being so careful about collective credit that you sound like you watched.
  Costs a level, and it's the far more common error among people who are actually good.

The target is neither modest nor inflated. It's **precise**.

---

## 1. The attribution sentence

A reusable structure that is honest and still claims properly:

> *"The team was running X. My piece was Y. The specific call I made was Z, and here's what
> moved."*

Worked example:

> *"The team was training a 70B dense model on 1,024 H100s. I owned the data pipeline —
> dedup, filtering, and the loader. The specific call I made was replacing document-level
> exact-hash dedup with MinHash+LSH at a 0.8 Jaccard threshold, which cut about 11% of the
> corpus that exact matching had missed. I also rebuilt the loader to be resumable to the exact
> sample, which is what let us diagnose a loss spike at 41k by replaying the same batches —
> that turned out to be a corrupted shard rather than an optimizer problem."*

That paragraph does four things at once: it names the scope, names the ownership, names a
decision with a threshold you can defend, and demonstrates a downstream consequence. It is
also entirely honest about what was the team's and what was yours.

---

## 2. The three-levels-down rule

**Only claim what you can go three levels deep on.** Interviewers at this level probe
relentlessly, and the probe is the real test.

```
Claim:   "I switched us to MinHash+LSH for dedup."
Level 1: "Why LSH rather than pairwise Jaccard?"     → tractability at trillion-token scale
Level 2: "How did you pick 0.8?"                     → precision/recall on a labeled sample
Level 3: "What did it do to the loss curve?"         → and if you don't know, say so
```

If you cannot survive level 3, either don't raise it or raise it with the boundary marked:
*"I owned the implementation; the threshold came from a study the eval team ran, and I can
tell you how they set it up but not the details of their labeling."* **That sentence costs you
nothing and buys enormous credibility.** Hedging vaguely costs you a lot.

---

## 3. What separates the levels

| Level | The story sounds like |
|---|---|
| **Mid** | "I implemented X, it worked, here's how it works." Execution on a defined task. |
| **Senior** | "I owned X. I chose A over B because of constraint C, and here's what it cost us." Ownership of a decision with an articulated trade. |
| **Staff+** | "I noticed the thing we were optimizing was wrong. Here's how I got the team to re-scope, and what the second-order effects were." Judgment about what work should exist. |

**The move up is never about a bigger model.** It's about how far back in the decision chain
your contribution sits — from executing a plan, to choosing within a plan, to changing the plan.

---

## 4. When you can't disclose

NDAs are normal here and interviewers expect them. The mistake is letting confidentiality turn
you into a vague candidate.

**What you generally cannot say:** exact parameter counts, exact data mixtures, proprietary
architecture details, unreleased results.

**What you can almost always say:** the *shape* of the problem, the tradeoff space, your
reasoning, the general class of technique, and orders of magnitude.

Rewrite: ~~"we trained a 400B model on 4,096 H100s with a 3.2% code fraction"~~ →
*"a dense model in the hundreds of billions, on a few thousand GPUs, and one thing I'd
highlight is how much the code fraction mattered for non-code reasoning — that surprised me,
and it changed how I'd weight a mixture next time."*

Same substance, no disclosure. **And say the boundary out loud** — "I can talk about the
reasoning but not the numbers" — rather than silently going fuzzy. Stated boundaries read as
professional; unstated ones read as thin.

---

## 5. Questions to have ready

- **"Walk me through your last run."** Scope, your piece, one decision, one thing that broke.
- **"What would you do differently on that run?"** Have a real answer. "Nothing" is a failing
  answer at any level.
- **"What broke, and how did you find it?"** Your best story. Debugging narratives are where
  pretraining depth is most legible — see the C04 and C12 protocols for the shape.
- **"What's the most surprising thing you learned?"** Genuinely useful signal for taste.
- **"What do you not know about pretraining?"** Naming a real gap and how you'd close it is a
  strength. Claiming there isn't one is a red flag.
- **"Why leave a pretraining team?"** Prepare this carefully — it's the one they'll wonder
  about, and it should never be a complaint.

---

## 6. The cross-track version

If you're also interviewing for FDE roles, your pretraining background is a **differentiator**
and should be framed as one, not apologized for:

> *"I've spent 18 months on the training side, so when a customer asks why inference costs
> what it does, I can explain it from the hardware up rather than from the pricing page."*

The thing to avoid is sounding like FDE is your fallback. Frame it as range: you understand
the model, not just the API — and that's exactly the gap most applied engineers have.

---

## Self-check

1. Give your attribution sentence for your last run. Does it separate team from you?
2. Name one claim you'd make and take it three levels down. Where do you run out?
3. What would you do differently on your last run? Be specific.
4. Rewrite your project summary to be NDA-safe without becoming vague.
5. What don't you know about pretraining, and how would you close it?
6. Which of the three levels does your default story currently sound like?
