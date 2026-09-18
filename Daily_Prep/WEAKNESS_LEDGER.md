# 📊 Weakness Ledger

Running scorecard. Updated after every graded answer. This drives question selection —
weak topics get re-asked under different framings until they hit two consecutive 3+ grades.

**Last updated:** 2026-09-18 (Day 1 — baseline, no data yet)

> **Open calibration question:** 18 months of pretraining covers a wide range — from
> "owned the data pipeline" to "owned the model architecture" to "owned the 4096-GPU run."
> Early questions probe which, since it determines whether the pretraining gap is modeling
> depth or systems depth. Do not assume either until graded.

---

## Active Weak Areas

*Empty — baseline diagnostic in progress. Topics populate here as answers get graded.*

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

*Populates after ~5 graded answers.*
