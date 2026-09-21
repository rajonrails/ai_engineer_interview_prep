# C12 — Checkpointing, Fault Tolerance, and MFU Debugging at Scale

> **Day 3, Evening** · Track: Pretraining · ⭐⭐⭐⭐
> **Read time:** ~25 min. **Why now:** this is the follow-up C02 promised — *"a node dies at
> step 40,000 of 100,000. Walk me through the next 20 minutes."* It's the most reliable
> separator between people who have run large jobs and people who have read about them,
> because nothing here is in a paper. It's all operational.

---

## The One Idea

**At scale, hardware failure is not an exception path — it is the steady state.** A run on
thousands of GPUs will be interrupted many times before it finishes. The engineering question
isn't "how do we avoid failures," it's **"what fraction of our compute do we lose to them,
and how do we drive that number down?"**

Frame your answer as an optimization over wasted compute and you sound like someone who has
owned a run. Frame it as "we save checkpoints" and you don't.

---

## 1. The failure math

Individual GPUs are reliable. Thousands of them, plus NICs, switches, storage, and power, are
not — failure rate scales with component count, so system MTBF falls roughly linearly as you
add nodes. Published large-run reports put real-world interruption rates on the order of
**several failures per day** on ten-thousand-GPU clusters. At that rate a 54-day run is
interrupted hundreds of times.

**Young/Daly optimal checkpoint interval:**

```
τ_opt ≈ √(2 · δ · M)        δ = checkpoint cost, M = mean time between failures
```

Worked example — 70B model, 1024 GPUs:

```
Model state:      70B × 16 bytes           = 1.12 TB
Checkpoint write: 1.12 TB ÷ ~10 GB/s       ≈ 112 s ≈ 0.03 h
Cluster MTBF:                              ≈ 10 h
τ_opt = √(2 × 0.03 × 10) = √0.6            ≈ 0.77 h ≈ every 46 minutes
```

Expected loss per failure is **half the interval** (~23 min of compute) plus restart overhead.
Being able to produce this derivation on a whiteboard is the answer to "how often do you
checkpoint?" — the wrong answer is a number with no reasoning attached.

Note what the formula implies: **halving checkpoint cost lets you checkpoint √2 times more
often at the same overhead.** That's why the engineering below is worth it.

---

## 2. Making checkpoints cheap

| Technique | Mechanism | Effect |
|---|---|---|
| **Distributed checkpointing** | Each rank writes its own shard in parallel; no gather to rank 0 | Turns a serial 1.12 TB write into a parallel one. Table stakes. |
| **Asynchronous checkpointing** | Copy state to pinned CPU memory (fast), write to storage in the background while training continues | Blocking time drops to the device-to-host copy — often 10x or better |
| **In-memory / peer replication** | Replicate state to a peer node's memory as the fast-recovery path; persistent storage is the slower backstop | Recovery from the common single-node failure without touching storage at all |
| **Sharded optimizer state** | You're already sharding under FSDP/ZeRO — write it sharded too | Avoids reconstructing full state just to serialize it |
| **Tiered retention** | Frequent checkpoints to fast local NVMe, occasional ones to durable object storage | Cost control; you don't need every checkpoint to survive forever |

**The failure mode to name:** synchronous checkpointing to a shared filesystem, where every
rank writes at once and saturates the storage network. Your MFU plot shows a periodic notch
with exactly the checkpoint period. That notch is the diagnostic.

---

## 3. What a checkpoint must contain

Getting this wrong produces bugs that don't surface for days.

- Model parameters, optimizer state (m, v), LR scheduler state, and step count — obvious.
- **RNG state per rank** — for dropout and any stochastic path.
- **Data loader position, exactly.** Not "we were at step 40,000" but *which samples had been
  consumed*. Many pipelines resume at the epoch or shard boundary and silently re-show or skip
  data. It corrupts reproducibility, and it makes the C04 loss-spike diagnostic ("rewind and
  replay the same batches") impossible to run.
- **The full config** — code version, hyperparameters, parallelism layout. A checkpoint you
  can't reproduce the environment for is a liability.

---

## 4. The 20-minute answer

*"A node dies at step 40,000 of 100,000."*

1. **Detect fast.** A dead rank makes collectives hang, not crash — the job sits at 0% until a
   timeout. Set aggressive NCCL timeouts and a heartbeat; otherwise you discover it an hour
   later. This detail alone signals real experience.
2. **Automate the restart.** Job schedulers should relaunch from the last checkpoint without a
   human. If a person has to notice and type something, your effective MTBF includes their
   response time and their sleep schedule.
3. **Hot spares.** Keep a few idle nodes in the allocation so a replacement doesn't wait on
   the cluster queue.
4. **Elastic training** where supported — continue at reduced world size rather than halting,
   adjusting gradient accumulation to preserve global batch size.
5. **Quarantine the node.** Return it to the pool untested and you'll hit it again in twenty
   minutes.
6. **Check whether it was really dead.** Which brings us to the harder case.

---

## 5. Stragglers and silent corruption — the nasty failures

**Stragglers.** A GPU that is slow but alive is worse than a dead one. Collectives run at the
speed of the slowest participant, so **one degraded GPU throttles all 1024.** Causes: thermal
throttling, ECC error correction overhead, a degraded NIC, a bad cable, or another tenant on
a shared node. Detection: per-rank step timing, and per-rank time-spent-in-collectives —
the slow rank is the one *not* waiting while everyone else is.

**Silent data corruption.** A GPU computing wrong results without erroring is real and
documented at scale. Detection: periodic deterministic self-checks, checksums on gradients,
or cross-replica comparison of a known computation. Symptom: unexplained loss spikes that
follow a specific node rather than specific data — which is exactly why the C04 protocol says
to log data indices *and* correlate with hardware.

---

## 6. Debugging MFU drops

*"MFU fell from 45% to 28% overnight, same config."* Work it in this order:

1. **Is it one rank or all of them?** Per-rank step time answers it immediately. One slow rank
   → straggler, go to §5. All ranks → systemic.
2. **Is it periodic?** Overlay the checkpoint interval. A notch at exactly that period is
   checkpoint I/O blocking.
3. **Is the data loader keeping up?** Time spent waiting for batches. A saturated storage
   backend or too few loader workers starves the GPUs, and it's a common cause people check
   last.
4. **Is communication overlapping?** Profile. An FSDP all-gather that stopped overlapping —
   because a shape changed, or the prefetch depth is wrong — costs exactly this kind of drop.
5. **Did the workload change?** Longer sequences raise attention's quadratic share; a data
   mixture shift can change the padding ratio.
6. **Is the hardware throttling?** Clock speeds, power caps, temperatures. Sometimes it really
   is the room's cooling.

---

## Self-check

1. Derive the optimal checkpoint interval for a 400B model on 2048 GPUs, 8-minute checkpoint
   cost, 6-hour MTBF. What's your expected loss per failure?
2. Why does asynchronous checkpointing help so much, and what exactly does it still block on?
3. Name four things a checkpoint must contain beyond weights and optimizer state.
4. Why is a straggler worse than a dead node?
5. A dead rank makes the job hang rather than crash. Why, and what do you configure?
6. MFU drops 15 points with no config change. Give your first three checks, in order.
