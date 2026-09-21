# C11 — Production Monitoring and Incident Response for LLM Systems

> **Day 3, Evening** · Track: FDE · ⭐⭐⭐⭐
> **Read time:** ~20 min. **Why now:** pairs with C03. Evals tell you whether a change is good
> *before* you ship; monitoring tells you what's happening *after*. The interview question is
> usually a disguised incident: "users say it got worse last Tuesday. Go."

---

## The One Idea

**LLM systems fail silently. Nothing throws.** A traditional service degrades loudly — 500s,
timeouts, saturated queues. An LLM system that has become subtly worse returns 200 OK, in
normal latency, with fluent, confident, wrong output. Your dashboards stay green while the
product degrades.

So the monitoring job is different: you are not watching for *errors*, you are watching for
*distribution shift* in things that have no exception to throw.

---

## 1. Log the trace, not the metric

Metrics you didn't think to record in advance can't be recovered. Log full traces and compute
metrics later.

Per request: request ID, tenant, model **and version**, prompt **version**, full input and
output, token counts (in/out/cached), **TTFT and TPOT separately** (per C01 — they have
disjoint causes), retrieved chunk IDs and scores, every tool call with args and result,
finish reason, cost, and any user feedback signal that arrives later.

Two of these are the ones people skip and then regret: **prompt version** and **retrieved
chunk IDs**. Without the first you cannot correlate a regression with a deploy. Without the
second you cannot tell a retrieval failure from a generation failure after the fact, which is
exactly the distinction C07 says everything depends on.

---

## 2. Three tiers of signal

| Tier | Metrics | Alerting |
|---|---|---|
| **System** | Latency p50/p95/p99 split into TTFT and TPOT, error rate, provider 429s/5xx, throughput, queue depth | Page. Conventional thresholds work. |
| **Behavioral proxies** | Refusal rate, mean output length, tool-call error rate, retry rate, conversation turn count, fallback-model usage, empty-retrieval rate | **Alert on distribution shift, not absolute value.** These are your early-warning system. |
| **Quality & business** | Task completion, escalation-to-human rate, thumbs, abandonment, cost per completed task | Dashboard and review. Too slow and noisy to page on. |

**The tier-2 insight worth stating in an interview:** refusal rate and mean output length are
cheap, real-time, and remarkably sensitive. A prompt change that broke something usually moves
one of them within minutes — long before a human reports anything. Watching them costs
nothing and buys you hours.

---

## 3. What actually causes production regressions

| Cause | How you detect it |
|---|---|
| **Silent model change** | Provider updates a floating alias under you. **Pin exact model versions** — this is the cheapest incident prevention there is. |
| **Prompt deploy** | Correlate the metric break with prompt version. Trivial if you logged it, near-impossible if not. |
| **Retrieval index staleness** | Empty-retrieval rate climbing, mean retrieval score drifting down. Alert on index age directly. |
| **Input drift** | New user segment, new language, new query shape. Cluster embeddings of incoming queries and watch cluster mass move. |
| **Context growth** | Conversations get longer over a release, cost and latency creep up, quality drops from mid-context dilution |
| **Provider degradation** | Latency and error rate move without any change on your side. This is why you keep a second provider wired up. |
| **Data or permissions change upstream** | Retrieval starts returning documents it shouldn't, or stops returning ones it should |

---

## 4. Continuous quality measurement

You cannot human-review production traffic at volume, so:

1. **Sample and judge.** Run an LLM judge (C03, with its biases and calibration) over a
   stratified sample of production traffic continuously. Track the score as a time series.
2. **Stratify the sample.** Overall averages hide the slice that broke. Sample by tenant,
   query type, and retrieval outcome.
3. **Keep a golden set in production.** A fixed set of requests replayed on a schedule against
   live infrastructure. Any movement is a real change, with user-traffic variation controlled
   out. This is the single highest-signal monitor you can build.
4. **Close the loop.** Every production failure becomes an eval row. Without this your eval
   set gets less representative every week while looking increasingly green.

---

## 5. The incident playbook

*"Users say it got worse last Tuesday."*

1. **Establish the delta.** What changed Tuesday — prompt, model, index, retrieval config,
   traffic mix? Your change log answers this in a minute if you keep one.
2. **Reproduce on the golden set.** Replay Monday's version vs. today's on identical inputs.
   Movement → your change. No movement → traffic or provider.
3. **Decompose.** Retrieval or generation? Compare retrieval metrics across the boundary
   before touching a single prompt.
4. **Roll back first, diagnose second.** Prompts and configs should be revertible in minutes.
   Restoring service is not the same task as understanding the failure; do them in that order.
5. **Check the boring things.** Provider status page, index age, credential and quota expiry,
   a silently upgraded SDK.
6. **Write the eval row.** Then the incident can't recur unnoticed.

**The reflex to demonstrate:** a change log and pinned versions turn an open-ended
investigation into a bisect. Most LLM incidents are hard only because nobody recorded what
changed.

---

## Self-check

1. Why are LLM regressions harder to detect than conventional service failures?
2. Name two cheap real-time proxies that catch quality regressions within minutes, and why
   they work.
3. What is a production golden set and why does it beat sampled traffic for detecting change?
4. "It got worse last Tuesday." First four things you check, in order.
5. Why log retrieved chunk IDs, specifically?
6. Your provider silently updates a model alias. What breaks, and what should have prevented it?
