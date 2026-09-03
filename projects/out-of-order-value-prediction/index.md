---
layout: page
title: "Out-of-Order RISC-V Core Model with Value Prediction"
permalink: /projects/out-of-order-value-prediction/
---

# Out-of-order RISC-V core model with value prediction

I extended a cycle-accurate C++ model of an 8-wide out-of-order RISC-V core — roughly 16,000 lines — with a complete stride value prediction subsystem, then proved it correct against an architectural reference model on every committed instruction.

Value prediction is a strange idea the first time you meet it. Branch prediction guesses *where* control flow goes. Value prediction guesses *what a result will be*, hands that guess to dependent instructions before the producer has executed, and breaks the data dependency outright. When it works, a chain of dependent operations stops being a chain.

## Core configuration

| Structure | Size |
| --- | --- |
| Fetch / dispatch / issue / retire width | 8 |
| Pipeline depth | 10 stages |
| Active list (reorder buffer) | 256 entries |
| Issue queue | 32 entries |
| Load and store queues | 32 entries each |
| Physical register file | 320 registers |
| Branch checkpoints | 32 |
| Value prediction queue | 64 entries |

## The stride predictor

Each predictor entry is indexed by program counter and holds the last retired value, the observed stride, a confidence counter, and an instance count. A prediction is simply `retired_value + instance x stride`.

Confidence is what keeps this from being reckless. The counter increments only when a newly observed stride matches the previous one, and resets to zero the moment it does not. Only entries at maximum confidence are allowed to predict. An instruction whose result changes unpredictably therefore trains its entry to silence rather than to noise.

## Where the pipeline touches it

The subsystem spans four stages, and getting the ordering right is most of the work:

| Stage | Action |
| --- | --- |
| **Rename** | Look up the predictor, allocate a value prediction queue entry, and mark the instruction as using a prediction if the entry is confident |
| **Dispatch** | Write the predicted value into the physical register file and mark the destination ready, so dependants can issue immediately |
| **Execute** | Compare the real result against the prediction and flag a misprediction if they differ |
| **Retire** | Train the predictor from the queue head using the actual committed value, then pop the entry |

The value prediction queue itself is a circular buffer with phase bits to distinguish full from empty. Entries are allocated in program order at rename and consumed in program order at retire, which is what lets the predictor train on committed values rather than speculative ones.

## Recovery is the hard part

Predicting is easy. Being wrong safely is not.

When a confident prediction turns out to be incorrect, the pipeline squashes everything after the offending instruction once it commits. That much is conventional. The subtlety is that the value prediction queue and the predictor's instance counters are *also* speculative state, and they have to be rolled back in step with everything else.

Branch mispredictions make this sharper still. On a branch squash, the queue tail is restored from the branch checkpoint, entries belonging to squashed instructions are invalidated, and the per-entry instance counters are decremented so the predictor does not believe it has seen values that never architecturally happened. Getting speculative and committed state to stay coherent through recovery taught me more about out-of-order execution than any lecture did.

## Proving it correct

Every committed instruction is checked against a Spike-based architectural reference model. Spike is the official RISC-V ISA simulator: it models architectural state exactly and knows nothing about timing, which makes it an ideal oracle.

The functional simulator runs ahead and records each instruction's architectural effects. At retire, the timing model pops the matching record and compares the instruction PC, both source register values, the destination value, load and store addresses, atomic and store-conditional results, and control and status register state. Any divergence halts the simulation immediately with the instruction number and cycle.

This is the mechanism that makes the whole project trustworthy. The checker does not care that the core executed out of order, speculated down a branch, or invented a value at dispatch — only that the retired architectural state is exactly what the ISA says it should be. A value prediction that escaped without correct recovery would surface instantly as a register mismatch. The technique is usually called lockstep co-simulation, or step-and-compare, and it is standard practice in CPU verification.

## Context

Built for ECE 721 (Advanced Microprocessor Architecture) at NC State University, taught by Prof. Eric Rotenberg. The out-of-order framework and its checker infrastructure were provided by the course; the value prediction subsystem across rename, dispatch, execute and retire was my contribution.
