---
layout: page
title: "Out-of-Order RISC-V Core Model with Value Prediction"
permalink: /projects/out-of-order-value-prediction/
---

# Out-of-order RISC-V core model with value prediction

I implemented stride value prediction inside a cycle-accurate C++ model of an out-of-order RISC-V core, then designed an original extension — a squash-triggered selective filter — that lifted harmonic-mean IPC across 15 SPEC benchmarks from **1.658 to 2.032, a 22.5% improvement**, while staying 10 KB under the storage budget.

Value prediction is a strange idea the first time you meet it. Branch prediction guesses *where* control flow goes. Value prediction guesses *what a result will be*, hands that guess to dependent instructions before the producer has executed, and breaks the data dependency outright. When it works, a chain of dependent operations stops being a chain.

## Baseline: stride predictor and value prediction queue

Each predictor entry is indexed by program counter and holds five fields: a tag, a saturating confidence counter, the last retired value, the observed stride, and an in-flight instance count. The prediction is

```text
pred = retired_value + (instance + 1) x stride
```

Confidence is what keeps this from being reckless. At retire, if the newly computed stride matches the stored one the counter increments; on any mismatch it resets to zero. Only entries at maximum confidence may predict, so an instruction whose result changes unpredictably trains its entry into silence rather than into noise.

The value prediction queue carries predictions through the pipeline in program order:

| Stage | Action |
| --- | --- |
| **Front end** | On a tag hit with a confident entry, push the index and prediction into the queue and increment the instance count |
| **Rename** | Broadcast the predicted value on the result bus so dependants wake early |
| **Execute** | Compare the computed value against the prediction; flag a mismatch but let the pipeline continue |
| **Retire** | Pop the queue in order and train the predictor from the *computed* value, never from the prediction |
| **Recovery** | On a retire-time value misprediction, squash the entire pipeline |

Training from the computed value rather than the prediction matters more than it looks. Train from your own guess and the predictor becomes a feedback loop that confirms whatever it already believed.

## The problem I found

The specified recovery scheme detects a value misprediction at **retire**. But the misprediction is known at **execute**, roughly 45 cycles earlier.

During that window the predictor entry is still saturated at maximum confidence, so every subsequent fetch of the same program counter predicts again — and predicts wrong again. A tight loop can issue a storm of wrong predictions from a single entry before the first one even reaches retire and resets the confidence counter.

That asymmetry is the whole problem. A *missed* prediction costs one lost opportunity. A *wrong* prediction costs a full pipeline squash, roughly an order of magnitude more. The baseline design spends the entire execute-to-retire gap manufacturing expensive mistakes it already knows about.

## My extension: a squash-triggered selective filter

Three small counters per entry, 24 bits in total, that gate the *consumer* side without touching the training rules at all:

| Field | Bits | Behaviour |
| --- | --- | --- |
| `penalty` | 8 | Cooldown set the moment execute detects a misprediction, decremented on each lookup. While non-zero, this entry is forbidden from predicting |
| `bad_streak` | 8 | Increments on each new misprediction and escalates the cooldown exponentially — 64, 128, 255, 255 — so a chronically wrong PC is silenced for longer each time |
| `good_streak` | 8 | Increments on each correct retire; at 32 or more, the entry is allowed to predict one confidence level early, rewarding a demonstrably stable PC |

The design deliberately intervenes at the point of *use* rather than at training. Every mandatory rule survives untouched: the five specified fields, retire-time in-order queue-driven training, training from computed values, confidence reset on stride mismatch, unchanged eligibility logic, and full-pipeline squash recovery. To the specification, a filtered prediction is indistinguishable from an entry that simply chose not to predict.

Storage came to 176 bits per entry across 1024 entries — **22 KB against a 32 KB cap**, of which my three fields account for 3 KB.

## Results

Fifteen SPEC CPU2006 and CPU2017 benchmarks, 10 million instructions each:

| Benchmark | Baseline IPC | With filter | Change |
| --- | --- | --- | --- |
| 429.mcf | 0.82 | 1.68 | +105% |
| 623.xalancbmk_s | 1.05 | 1.86 | +77% |
| 649.fotonik3d_s | 0.73 | 1.07 | +47% |
| 434.zeusmp | 2.04 | 2.30 | +13% |
| 473.astar | 2.59 | 2.89 | +12% |
| 482.sphinx3 | 1.62 | 1.76 | +9% |
| 464.h264ref | 5.46 | 5.78 | +6% |
| 400.perlbench | 5.01 | 5.28 | +5% |
| 456.hmmer | 2.36 | 2.40 | +2% |
| 471.omnetpp | 0.82 | 0.83 | +1% |
| 641.leela_s | 2.38 | 2.40 | +1% |
| 603.bwaves_s | 1.80 | 1.81 | +1% |
| 401.bzip2 | 2.70 | 2.71 | 0% |
| 458.sjeng | 3.16 | 3.14 | −1% |
| 453.povray | 3.12 | 3.09 | −1% |
| **Harmonic mean** | **1.658** | **2.032** | **+22.5%** |

The distribution is more interesting than the average. Every large win is on a **low-IPC, memory-bound** benchmark: mcf at 0.82, xalancbmk at 1.05, fotonik3d at 0.73. These are programs dominated by dependent load chains, where the machine is starved of independent work and a correctly predicted value breaks a chain that was blocking everything behind it.

The benchmarks that gain nothing are the ones already running well. h264ref at 5.46 IPC and perlbench at 5.01 are extracting plenty of instruction-level parallelism on their own; there is no stalled dependence chain for a prediction to unblock. And povray and sjeng go very slightly *negative*, because occasional mispredictions cost squashes that the small number of correct predictions does not repay.

Value prediction, in other words, is not a general speedup. It is a targeted remedy for dependence-limited code, and the harmonic mean — the correct average for a rate like IPC — hides that entirely.

## Verification

Every committed instruction is checked against a Spike-based architectural reference model. Spike is the official RISC-V ISA simulator: it models architectural state exactly and knows nothing about timing, which makes it an ideal oracle.

The functional simulator runs ahead and records each instruction's architectural effects. At retire, the timing model pops the matching record and compares the instruction PC, both source register values, the destination value, load and store addresses, atomic and store-conditional results, and control and status register state. Any divergence halts the simulation immediately with the instruction number and cycle.

This is what makes the whole project trustworthy. The checker does not care that the core executed out of order, speculated down a branch, or invented a value at dispatch — only that retired architectural state is exactly what the ISA says it should be. A prediction that escaped without correct recovery would surface instantly as a register mismatch.

## Notes for revisiting this

### The one-sentence version

I found that the specified recovery scheme leaves a 45-cycle window in which a known-bad predictor entry keeps firing, added three counters to gate the consumer side during that window, and got 22.5% harmonic-mean IPC for 3 KB of extra storage.

### Why the filter works

Not because it makes predictions more accurate — it makes strictly *fewer* predictions. The gain comes from removing predictions that were already known to be wrong. The asymmetry between a missed prediction (one lost opportunity) and a wrong prediction (a full squash) means suppressing a doubtful prediction is nearly free, while allowing a wrong one is very expensive.

### Why exponential backoff rather than a fixed cooldown

A PC that mispredicts once may have hit a one-off irregularity and deserves another chance soon. A PC that mispredicts repeatedly is genuinely unpredictable and should be silenced for progressively longer. Doubling — 64, then 128, then saturating at 255 — separates the two cases without needing to classify them.

### Why `good_streak` exists

Purely to avoid over-penalising. Without it the filter is one-directional and a PC that recovers stable behaviour still has to climb all the way back to full confidence. Letting a demonstrably stable entry predict one level early buys back some of what the cooldown costs.

### Recovery, in detail

On a value misprediction the pipeline squashes everything after the offending instruction once it commits. On a *branch* squash — a different and more frequent event — the queue tail is restored from the branch checkpoint, entries belonging to squashed instructions are invalidated, and per-entry instance counters are decremented so the predictor does not believe it observed values that never architecturally happened. The prediction queue and the instance counters are speculative state and must be rolled back in step with everything else. This is the part that is genuinely hard to get right.

### Questions worth being ready for

**Why harmonic mean rather than arithmetic?** IPC is a rate. Averaging rates arithmetically over-weights the fast benchmarks; the harmonic mean corresponds to total instructions over total cycles, which is what actually matters.

**Why not just detect at execute and recover there?** That is a different and more invasive design — selective replay rather than full squash. The specification fixed the recovery scheme, so the filter works within that constraint instead of changing it.

**Does the filter break the specification?** No, and that was a design goal. It never touches training, never changes eligibility, and never alters confidence semantics. A filtered prediction is indistinguishable from an entry declining to predict.

**Why do some benchmarks regress?** Because the filter cannot prevent the *first* misprediction, only the storm afterwards. On a benchmark with few predictable values, you pay squash costs for the initial mistakes without enough correct predictions to repay them.

### What this project does not cover

No selective replay or execute-time recovery, no context-based or hybrid value predictors (stride only), no multicore or coherence interaction, and no hardware implementation — this is a simulation study, so the storage accounting is analytical rather than synthesized.

## Context

Built for ECE 721 (Advanced Microprocessor Architecture) at NC State University, taught by Prof. Eric Rotenberg, as the Project 4 value-prediction competition entry. The out-of-order framework and its reference-model checker were provided by the course; the value prediction subsystem and the selective filter were my contribution.
