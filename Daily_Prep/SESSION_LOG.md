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

| # | Track | Topic |
|---|---|---|
| C07 | FDE | [RAG and Retrieval Quality](concepts/Day_02_Evening_RAG_Retrieval_Quality.md) |
| C08 | Pretraining | [Pretraining Data: Dedup, Filtering, Mixing, Annealing](concepts/Day_02_Evening_Pretraining_Data_Pipelines.md) |

### 📋 End of Day 2

**Questions asked:** 3 cumulative · **Answered:** 0 · **Graded:** 0

**No new question posed tonight — deliberate.** Three open and unanswered is already a
backlog; adding a fourth would make the queue less likely to be engaged with, not more.
The concept drops continue because they have standalone value, but the grading loop —
which is the actual mechanism of this program — has not started. The ledger cannot
function without answers, and topic selection stays coverage-driven rather than
weakness-driven until it does.

---

## Day 3 — 2026-09-20

### ☀️ Morning Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C09 | FDE | [Fine-Tuning: When, When Not, and What It Actually Costs](concepts/Day_03_Morning_Fine_Tuning_Decisions.md) |
| C10 | Pretraining | [Attention Variants, FlashAttention, and Long Context](concepts/Day_03_Morning_Attention_And_Long_Context.md) |

Selection rationale: coverage-driven (no graded data yet). C09 closes the loop on Q1, where
the customer asks to fine-tune for what is a retrieval problem. C10 covers transformer
internals and attention variants, both untested and both core to the pretraining loop.

### 🎤 Questions

No new question posed. Q1, Q2 and Q3 remain open from Days 1-2; the standing offer to
unstick the loop (model answer for Q2, shorter question format, or a format change) is
also open. Not re-asking.

### 🌙 Evening Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C11 | FDE | [Production Monitoring and Incident Response](concepts/Day_03_Evening_Production_Monitoring.md) |
| C12 | Pretraining | [Checkpointing, Fault Tolerance, and MFU Debugging at Scale](concepts/Day_03_Evening_Checkpointing_And_Fault_Tolerance.md) |

C11 pairs with C03: evals measure before shipping, monitoring measures after.
C12 delivers the follow-up C02 promised — the "node dies at step 40,000" scenario.

### 📋 End of Day 3

**Questions asked:** 3 cumulative · **Answered:** 0 · **Graded:** 0 · **Concepts delivered:** 12

Ledger remains at baseline. Topic selection is still coverage-driven.

---

## Day 4 — 2026-09-21

### ☀️ Morning Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C13 | FDE | [Scoping Under Ambiguity: The Core FDE Skill](concepts/Day_04_Morning_Customer_Scoping.md) |
| C14 | Pretraining | [Optimizers, Learning-Rate Schedules, and muP](concepts/Day_04_Morning_Optimizers_And_Schedules.md) |

Selection rationale: C13 is the last major untested FDE area and the one that is judgment
rather than knowledge — it is the round strong engineers most often fail. C14 completes the
pretraining core with the "you only get one run" hyperparameter question.

### 🎤 Questions

No new question. Q1, Q2 and Q3 remain open from Days 1-2.

### 🌙 Evening Concept Drop — 2 topics

| # | Track | Topic |
|---|---|---|
| C15 | FDE | [Context Engineering](concepts/Day_04_Evening_Context_Engineering.md) |
| C16 | Pretraining | [Post-Training and the Pretraining Handoff](concepts/Day_04_Evening_Post_Training_Handoff.md) |

### 📋 End of Day 4

**Questions asked:** 3 cumulative · **Answered:** 0 · **Graded:** 0 · **Concepts delivered:** 16

With C15 and C16, both coverage maps are essentially complete end to end. From here the
useful work is depth on weak areas and repetition under pressure — both of which require
graded answers. Continuing to add breadth has diminishing returns.

---

## Day 5 — 2026-09-22

### ☀️ Morning Drop — 2 worked answers (format change)

| # | Track | Topic |
|---|---|---|
| WA1 | FDE | [Worked Answer — Q1: Fintech cost and latency](worked_answers/Q1_Fintech_Cost_And_Latency.md) |
| WA2 | Pretraining | [Worked Answer — Q2: Loss spike at step 41,000](worked_answers/Q2_Loss_Spike_At_Step_41k.md) |

**Why the format changed.** Both coverage maps completed on Day 4, so additional breadth has
low marginal value. These deliver the standing Day 2 offer instead: full model answers at the
Strong Hire bar, annotated with what each move signals to an interviewer, plus a grade
rubric showing what separates a 2 from a 4 on the same question.

This also makes the grading bar concrete rather than abstract — useful whenever the loop
does start, and useful on its own as calibration.

### 🌙 Evening Drop
*Pending — fires 7pm PT*
