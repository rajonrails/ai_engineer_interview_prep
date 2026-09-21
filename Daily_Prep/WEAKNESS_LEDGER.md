# 📊 Weakness Ledger

Running scorecard. Updated after every graded answer. This drives question selection —
weak topics get re-asked under different framings until they hit two consecutive 3+ grades.

**Last updated:** 2026-09-21 morning (Day 4 — still no graded data; 3 questions open)

> **Open calibration question:** 18 months of pretraining covers a wide range — from
> "owned the data pipeline" to "owned the model architecture" to "owned the 4096-GPU run."
> Early questions probe which, since it determines whether the pretraining gap is modeling
> depth or systems depth. Do not assume either until graded.

---

## Active Weak Areas

*Empty. Q1 was posed but not answered, so nothing has been graded yet. This section stays
empty until there is real data — an unanswered question is not evidence of weakness, and
guessing at weak areas would make the ledger worse than useless.*

| Topic | Grades | Trend | Last tested | Notes |
|---|---|---|---|---|

---

## Retired (Proven Strong)

| Topic | Grades | Retired on |
|---|---|---|

---

## Coverage Map

Tracks which areas have been tested at all. Untested ≠ strong.

### Forward Deployed AI Engineer (50%)

| Area | Status | Best grade |
|---|---|---|
| Inference economics (prefill/decode, KV cache, batching) | ⬜ untested | — |
| RAG architecture & retrieval quality | ⬜ untested | — |
| Evals & measurement | ⬜ untested | — |
| Agents, tool calling, action safety | ⬜ untested | — |
| Context engineering & prompt discipline | ⬜ untested | — |
| Fine-tuning: when and when not | ⬜ untested | — |
| Production monitoring & incident response | ⬜ untested | — |
| Customer scoping under ambiguity | ⬜ untested | — |
| Live Python coding speed | ⬜ untested | — |
| Security: prompt injection, data boundaries | ⬜ untested | — |

### Pretraining Engineer (50%) — difficulty floor raised: 18mo experience, expect senior-bar questions

| Area | Status | Best grade |
|---|---|---|
| Transformer internals from scratch | ⬜ untested | — |
| Attention variants (MHA/MQA/GQA, flash attention) | ⬜ untested | — |
| Distributed training (DP/FSDP/TP/PP) | ⬜ untested | — |
| Mixed precision & numerical stability | ⬜ untested | — |
| Scaling laws & compute budgeting | ⬜ untested | — |
| Data pipelines, dedup, curriculum | ⬜ untested | — |
| Loss-curve debugging | ⬜ untested | — |
| GPU performance: MFU, memory, kernels | ⬜ untested | — |
| Checkpointing, fault tolerance, restart | ⬜ untested | — |
| Tokenizer design & vocab decisions | ⬜ untested | — |
| RoPE / positional schemes & context extension | ⬜ untested | — |
| Optimizer internals (AdamW, muP, LR schedules) | ⬜ untested | — |
| Post-training handoff (SFT/RLHF interface) | ⬜ untested | — |

---

## Recurring Patterns

Cross-cutting habits that cost points regardless of topic. The highest-leverage thing to fix.

*Populates after ~5 graded answers. Currently 0.*

**Process note (Day 2):** 8 concepts delivered, 0 answers graded. Reading is not the
bottleneck this program was built to address — the grading loop is. If the question format
is the friction (too long, too open-ended, wrong time of day), that is worth changing;
the format is a means, not the point.

---

## Concepts Delivered

Reading coverage. Separate from tested coverage above — reading a topic is not evidence of
being able to answer on it under pressure.

| # | Day | Track | Topic |
|---|---|---|---|
| C01 | 1 AM | FDE | Inference economics: prefill/decode, KV cache, arithmetic intensity |
| C02 | 1 AM | Pretraining | Parallelism strategy: DP/FSDP/TP/PP/EP, bubble math, comm volume |
| C03 | 1 PM | FDE | Evals: sample-size math, the eval ladder, judge biases, RAG decomposition |
| C04 | 1 PM | Pretraining | Mixed precision, loss spikes, numerical stability, debugging protocol |
| C05 | 2 AM | FDE | Agents: loop failure modes, tool design, action safety tiers, prompt injection |
| C06 | 2 AM | Pretraining | Scaling laws, 6ND, Chinchilla vs inference-aware budgeting |
| C07 | 2 PM | FDE | RAG: recall as ceiling, chunking, hybrid search + RRF, reranking, RAG vs long context |
| C08 | 2 PM | Pretraining | Data pipelines: MinHash dedup, quality filters, decontamination, mixing, annealing |
| C09 | 3 AM | FDE | Fine-tuning: the ladder, knowledge vs behavior, LoRA math, the customer conversation |
| C10 | 3 AM | Pretraining | MHA/MQA/GQA, FlashAttention online softmax, RoPE, context extension (PI/NTK/YaRN) |
| C11 | 3 PM | FDE | Monitoring: silent failure, trace logging, behavioral proxies, incident playbook |
| C12 | 3 PM | Pretraining | Young/Daly checkpoint interval, async/distributed checkpointing, stragglers, MFU debugging |
| C13 | 4 AM | FDE | Scoping: the discovery questions, finding the real constraint, two-week tests, the demo trap |
| C14 | 4 AM | Pretraining | AdamW decoupled decay, beta2=0.95, clipping placement, WSD schedules, muP transfer |
