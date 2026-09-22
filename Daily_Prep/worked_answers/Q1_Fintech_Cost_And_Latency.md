# Worked Answer — Q1: "It's slow and it costs too much, we want to fine-tune Haiku"

> **Track:** FDE · ⭐⭐⭐⭐ · Graded at the **Strong Hire (4)** bar
> **Read this as:** what a strong candidate says, *plus* what each move signals. The reasoning
> is the point, not the wording.

---

## The question

Series-C fintech, support assistant, six weeks in production. VP of Engineering:

> *"It's too slow and it's costing us way too much. We're at $180K/month and users are
> complaining. We were told a smaller model would fix both — we want to fine-tune Haiku on our
> tickets. Can you help us scope that?"*

Dashboard: **2.4M requests/month · median input 11,400 tokens · median output 95 tokens ·
p50 4.1s / p99 22s · every request carries the same 6,000-token system prompt.**

---

## Step 1 — Do the arithmetic before you say anything

The four numbers are the whole answer. Compute the token volumes:

```
Input tokens/month:   2.4M × 11,400  =  27.4B
Output tokens/month:  2.4M ×     95  =   0.23B

Input share of all tokens:  27.4 / 27.63  =  99.2%
```

**This is an input-dominated workload by a factor of 120:1.** Output tokens are priced higher
per token, but there are almost none of them. Whatever is driving $180K/month, it is the input
side.

Now decompose the input:

```
Static system prompt:  6,000 × 2.4M  =  14.4B tokens/month
Variable content:      5,400 × 2.4M  =  13.0B tokens/month

The same 6,000 tokens account for ~53% of all input tokens processed.
```

**Over half their bill is re-processing an identical prefix 2.4 million times.**

> **Signal:** doing this decomposition live, in under two minutes, from four dashboard numbers.
> Most candidates start proposing architecture. The numbers tell you what to propose.

---

## Step 2 — Test the premise against the arithmetic

Fine-tuning Haiku reduces the *price per token*. It does not reduce the *number of tokens*.
They would still process 27.4B input tokens every month, they would now own a model, and they
would have spent a quarter to get a discount they could have had in a week.

It also may not help latency the way they expect: with only 95 output tokens, decode is a
small share of the 4.1s. Most of that time is **prefill of an 11,400-token prompt** — which is
compute-bound and scales with prompt length, not model size, as strongly as they assume.

**But do not say "you're wrong."** Someone on their team already advocated for this, possibly
publicly. Say the arithmetic instead and let it do the work.

> **Signal:** rejecting the premise *on evidence* rather than on principle. "We never
> fine-tune" is dogma. "Fine-tuning changes cost-per-token and 99% of your tokens are input"
> is engineering.

---

## Step 3 — What you actually say on the call

> *"Before we scope a fine-tune, can I show you something from your own dashboard? About 99%
> of the tokens you're paying for are input, not output — you're sending 11,400 tokens per
> request and getting 95 back. And more than half of that input is the same 6,000-token system
> prompt, re-sent 2.4 million times a month.*
>
> *Fine-tuning a smaller model lowers your price per token. It doesn't lower how many tokens
> you send, and that's what's driving the bill. So I think there's a much cheaper thing to try
> first — and if it doesn't get you there, we'll have real numbers to justify the fine-tune.*
>
> *Can I ask a few questions to make sure I'm reading this right?"*

Three things that construction does: it uses their data, it makes the fine-tune contingent
rather than refused, and it ends by handing the floor back.

---

## Step 4 — The questions

1. **Are you using prompt caching?** If no, that's the headline. If yes, what's the hit rate,
   and is anything variable — a timestamp, a user name, a session ID — sitting in the prefix
   and invalidating it?
2. **What's in the 5,400 variable tokens?** Almost certainly retrieval. What's the top-k? Has
   anyone measured whether k=20 beats k=5 on answer quality, or was it picked once and never
   revisited?
3. **Are you streaming?** With 95 output tokens, streaming changes perceived latency
   dramatically for zero backend work.
4. **What's the p99 story?** 22s against a 4.1s median is a 5x spread. That's a different bug —
   queueing, a long-tail input size, or retries — and it's probably what users are actually
   complaining about. **Medians don't generate complaints; tails do.**
5. **Which complaint is real?** Is the CFO unhappy about $180K, or are users unhappy about
   latency? These have different fixes and you should know which one you're being hired for.

> **Signal:** question 4. Noticing that p99/p50 = 5x is a separate problem from the cost
> problem, and that the user complaints likely live in the tail, not the median.

---

## Step 5 — The proposal

**Week 1 — instrument and cache.**
- Turn on prompt caching for the 6,000-token prefix; audit it for anything variable.
- Add streaming if absent.
- Split the latency metric into TTFT and TPOT. You cannot fix what you haven't separated.
- Log p99 traces specifically, and find out what's different about them.

**Week 2 — trim the input.**
- Measure retrieval recall@k on 100 real questions. If recall@5 ≈ recall@20, drop k and remove
  thousands of tokens per request at no quality cost.
- Audit the system prompt. Six-thousand-token prompts are grown by accretion and usually
  contain contradictory or obsolete instructions.

**Then re-measure, and revisit the model question with real numbers.** Cheaper input may make
the current model affordable. If it doesn't, you now have an eval set — built in week 2 — and
a genuine case for a smaller model, with the infrastructure to verify it didn't regress.

**Rough expectation, stated as an expectation and not a promise:** caching addresses the ~53%
that is pure prefix re-processing, and retrieval trimming attacks a meaningful slice of the
rest. That is a config-and-tuning change measured in days, against a fine-tune measured in a
quarter.

> **Signal:** sequencing by cost-to-try, and naming what evidence would change your mind.
> Also: the eval set is a deliverable in its own right — they need it for the fine-tune anyway,
> so building it first is not a detour.

---

## What separates the grades

| Grade | What it looks like |
|---|---|
| **1 — No hire** | Starts scoping the fine-tune. Takes the request at face value. |
| **2 — Mixed** | Says "fine-tuning probably won't help" but can't say why from the numbers; suggests caching as one option among several without ranking. |
| **3 — Hire** | Identifies the input/output asymmetry and the static prefix, proposes caching first, asks good questions. |
| **4 — Strong hire** | All of the above, *plus* notices the p99/p50 gap is a separate problem, splits TTFT from TPOT before proposing latency fixes, frames the eval set as a prerequisite they need regardless, and handles the fine-tuning premise without making anyone wrong. |

---

## The transferable pattern

1. **Decompose the metric before proposing anything.** Total cost → input vs output. Total
   latency → TTFT vs TPOT. Averages → p50 vs p99.
2. **Test the stated solution against the arithmetic**, not against your priors.
3. **Sequence by cost-to-try**, cheapest first.
4. **Name the evidence that would change your mind** — it's what makes the recommendation
   trustworthy rather than merely confident.
