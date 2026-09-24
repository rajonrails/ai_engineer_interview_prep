# FDE Drill Bank

> Rapid-fire self-test. Cover the answer, say yours out loud, then check.
> **Out loud matters** — recognizing an answer and producing one are different skills, and
> interviews test the second.
> Mark anything you fumble and re-drill it in two days.

---

## Inference economics

**Q. Why do providers charge ~3x more for output than input tokens?**
Prefill processes all prompt tokens in parallel and is compute-bound. Decode is sequential and must stream every weight from HBM per token — memory-bandwidth-bound and far less efficient. Pricing reflects hardware, not strategy.

**Q. TPOT is 40ms, you need 20ms. Someone suggests doubling the batch size. What happens?**
Essentially nothing for that user. Batching raises throughput, not per-user decode speed — you're reading the same weights either way. Worse, it eats KV cache. TPOT needs a smaller model, quantization, speculative decoding, or better hardware.

**Q. Customer wants 200K context *and* high throughput on fixed hardware. Why can't they have both?**
HBM is split between weights and KV cache. `KV/token = 2 × layers × kv_heads × head_dim × bytes`, so context length and batch size compete for the same memory. Max batch ≈ (HBM − weights) / (KV per token × context).

**Q. When does speculative decoding *not* help?**
When you're already compute-bound — large batches, or heavy prefill. It exploits *spare compute* during bandwidth-bound decode. It also degrades when the draft model's acceptance rate is low.

**Q. Name the technique for each: GPU idles behind long requests / KV fragmentation / re-prefilling a stable prefix / one long prefill blocking everyone's decode.**
Continuous batching · PagedAttention · prefix caching · chunked prefill.

---

## Evals

**Q. 82% → 86% on n=100. Real improvement?**
No. `0.98/√100 ≈ ±10pp` at 95%. The intervals overlap almost completely. Use a paired design and report the CI on the *delta*.

**Q. Why is a paired design so much more sensitive?**
Per-example difficulty variance cancels — you count only disagreements (McNemar). Detects the same effect with roughly an order of magnitude fewer examples.

**Q. Three LLM-judge biases and their mitigations.**
Position → run both orderings, average; flip means tie. Verbosity → control for length in the rubric. Self-preference → judge with a different model family.

**Q. RAG end-to-end is 64%. What do you measure next, and why is the prompt last?**
Retrieval recall@k first — it's the ceiling. Then rank position, then chunk quality by reading them, then faithfulness. The prompt can't recover a chunk that was never retrieved.

**Q. Customer wants "95% accuracy" in the contract. Response?**
Ask: on what distribution, judged by whom, with what CI, and with what sample size. Accuracy without a defined eval set and an agreed grader isn't a commitment, it's a number. Propose the eval set as the first deliverable.

---

## Agents

**Q. Why is "never email more than 10 people" in the system prompt not a control?**
Prompts are probabilistic instructions, not enforcement. A `send_email` that raises on recipient #11 is a control. Prompts are not a security boundary.

**Q. Attacker files a support ticket with embedded instructions. Your agent reads tickets and has `get_credential`. Walk the compromise and the fixes, ranked.**
The agent reads attacker text as context and may comply. Fixes: (1) least privilege — that agent must not hold that credential; (2) reduced tool set while processing untrusted content; (3) human confirmation on tier 2–3 actions; (4) output filtering; (5) injection classifiers as defense in depth only.

**Q. Agent succeeds 80% of the time but uses 3x the necessary steps. Measure what, fix what?**
Trajectory efficiency and tool-selection accuracy. Usually tool design: too many tools, unclear descriptions, or errors returning stack traces instead of actionable guidance.

**Q. When is multi-agent actually justified?**
Different tool sets or permissions per sub-task (a real security argument), context-window pressure, genuine parallelism, or an independent critic with fresh context. Not "it feels organized."

**Q. What's the blast-radius formula?**
`auto-approve ceiling × rate limit per window`. It must hold when the model is fully captured, not merely buggy.

---

## RAG

**Q. Customer searches "E4021" and gets generic troubleshooting docs. Diagnose.**
Pure vector search. Embeddings don't know that token is special. Add BM25 and fuse with RRF; consider exact-match metadata extraction for identifier-shaped queries.

**Q. Why does RRF fuse on ranks rather than scores?**
Cosine similarity and BM25 scores are on incomparable scales with no principled normalization. Ranks are comparable by construction. `RRF(d) = Σ 1/(k + rank_i(d))`, k≈60.

**Q. Explain small-to-big chunking and the trade-off it resolves.**
Embed small chunks for precise matching, return the enclosing section for generation. Resolves small-chunks-retrieve-precisely-but-lack-context versus large-chunks-carry-context-but-dilute-the-embedding.

**Q. recall@50 is 94%, recall@5 is 61%. What do you build?**
A reranker. Retrieval is fine; ranking is the problem. Retrieve 50–100, cross-encode, keep 5.

**Q. 2M-token corpus, 1M-token model. Still build RAG?**
Yes — and the real answer is both. Cost and prefill latency scale with every token every call; access control needs filtering; citation needs provenance; freshness needs an index. But retrieve *broadly* into the large window rather than chasing top-5 precision.

---

## Fine-tuning

**Q. "Fine-tune on our 50K support tickets so it knows our product."**
That teaches the shape of tickets, not their contents, and what it memorizes is unverifiable. You'd get a more confident, no-more-correct model. Knowledge belongs in retrieval.

**Q. The one-sentence knowledge-vs-behavior test.**
"If the correct answer were pasted into the prompt, would the model get it right?" Yes → retrieval. No → maybe fine-tuning.

**Q. LoRA params at d_in=d_out=4096, r=32. Fraction of full FT?**
`32 × 8192 = 262,144` vs `16.78M` → **~1.6%**.

**Q. Why is multi-tenant data a *security* argument against fine-tuning?**
You cannot un-train one tenant's data out of a shared model. Retrieval with per-tenant filters is the only correct answer when users may see different data.

**Q. What do you eval after fine-tuning besides the target task?**
General benchmarks — catastrophic forgetting. A model that aces your classifier and lost everything else is a regression you'll ship blind.

---

## Monitoring and scoping

**Q. Two cheap real-time proxies that catch quality regressions within minutes.**
Refusal rate and mean output length. Both move fast when a prompt or model change breaks something, hours before anyone files a complaint.

**Q. Why log retrieved chunk IDs specifically?**
Without them you cannot separate a retrieval failure from a generation failure after the fact — and that distinction determines the entire fix.

**Q. "It got worse last Tuesday." First four checks.**
What changed Tuesday (change log) → replay the golden set on old vs new → decompose retrieval vs generation → roll back, *then* diagnose.

**Q. Why does "what happens if the AI gets it wrong?" constrain more than an accuracy target?**
It sets the architecture. Cheap error → ship and iterate. Expensive or regulated error → human in the loop, audit trail, possibly no LLM in the decision path at all.

**Q. Customer says "we want an AI that answers all our customer questions." First five questions.**
Walk me through the last time this went wrong · who does this today and what do they actually do · what happens if it's wrong · how will we know it worked · can I see ten real examples right now.
