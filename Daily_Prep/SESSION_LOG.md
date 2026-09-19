# 📓 Session Log

Chronological record of concept drops, questions, grades, and follow-ups.

---

## Day 1 — 2026-09-18

**Track set:** Dual — FDE (50%) / Pretraining Engineer (50%)
**Baseline:** Strong SWE + 18 months hands-on pretraining
**Revision:** Track opened from FDE-weighted to 50/50 mid-session once pretraining
experience was disclosed. Pretraining difficulty floor raised to senior bar.

### ☀️ Morning Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C01 | FDE | [Inference Economics: Prefill, Decode, and the KV Cache](concepts/Day_01_Morning_Inference_Economics.md) |
| C02 | Pretraining | [Choosing a Parallelism Strategy: DP, FSDP, TP, PP, EP](concepts/Day_01_Morning_Parallelism_Strategies.md) |

Why these first: C01 is the bottom of every FDE latency/cost conversation, and it's the
one area where pretraining experience does *not* transfer cleanly — training intuitions
about batching and memory mislead on the serving side. C02 is the single most-asked
pretraining interview topic, pitched at derive-it-from-scratch rather than recall level.

### 🎤 Questions

| # | Question | Area | Grade | Notes |
|---|---|---|---|---|
| Q1 | Customer's assistant is "slow and expensive" — diagnose and fix | Inference economics + scoping | ⏳ pending | Baseline diagnostic. Also probes pretraining↔inference transfer. |

### 🌙 Evening Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C03 | FDE | [Evals: Measuring Whether an LLM System Actually Got Better](concepts/Day_01_Evening_Evals_And_Measurement.md) |
| C04 | Pretraining | [Mixed Precision, Loss Spikes, and Numerical Stability](concepts/Day_01_Evening_Numerical_Stability.md) |

C03 pairs with C01: morning was "what does it cost," evening is "did it get better."
C04 pairs with C02: morning was "how do you split the work," evening is "what breaks at 3am."

### 📋 End of Day 1

**Questions asked:** 1 · **Answered:** 0 · **Graded:** 0

Q1 remains open. No grades recorded, so the ledger stays at baseline and tonight's topics
were selected by coverage priority rather than by demonstrated weakness. Topic selection
becomes weakness-driven as soon as there are graded answers to work from.

---

## Cadence

| When | What |
|---|---|
| 7:00am PT daily | 2 concept drops — 1 FDE + 1 pretraining |
| 7:00pm PT daily | 2 concept drops — 1 FDE + 1 pretraining |
| Throughout the day | Interview questions, one at a time, in the feedback loop |

Both drops are automated as scheduled routines bound to this prep session.

---

## Day 2 — 2026-09-19

### ☀️ Morning Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C05 | FDE | [Agents, Tool Calling, and Action Safety](concepts/Day_02_Morning_Agents_And_Action_Safety.md) |
| C06 | Pretraining | [Scaling Laws and Compute Budgeting](concepts/Day_02_Morning_Scaling_Laws.md) |

Selection rationale: still no graded answers, so topics are chosen by coverage priority
rather than demonstrated weakness. C05 and C06 are the two highest-frequency untested areas
on their respective coverage maps — agents are what FDEs actually deploy, and compute
budgeting is the canonical pretraining design question.

### 🎤 Questions

| # | Question | Area | Grade | Notes |
|---|---|---|---|---|
| Q1 | Fintech assistant "slow and expensive" — diagnose and respond to the VP | Inference economics + scoping | ⏳ open | Carried from Day 1 |
| Q2 | 70B loss spike at step 41k, $400K restart cost | Loss-curve debugging + judgment | ⏳ open | Carried from Day 1 |
| Q3 | Agent with refund authority — design the safety model | Agents + action safety | ⏳ open | |

### 🌙 Evening Concept Drop — 2 topics
*Pending — fires 7pm PT*
