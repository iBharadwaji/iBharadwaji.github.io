---
layout: page
title: "Branch Predictors and Dynamic Instruction Scheduling"
permalink: /projects/branch-prediction-and-scheduling/
---

# Branch predictors and dynamic instruction scheduling

Two C++ simulators modelling the structures that let a processor run ahead of its own dependencies. Both were characterized by sweeping their design spaces across SPEC traces rather than measured at a single configuration.

## Branch prediction

Three predictors of increasing sophistication, swept across table sizes from 128 entries to 1M entries (`m` from 7 to 20):

| Predictor | Mechanism |
| --- | --- |
| **Bimodal** | 2-bit saturating counters indexed by PC |
| **Gshare** | Counters indexed by PC hashed against an `n`-bit global history register, so one static branch predicts differently depending on the path taken to reach it |
| **Hybrid** | Bimodal and gshare in parallel, with a chooser table of 2-bit counters selecting per branch |

### Where bimodal saturates

| Benchmark | Table size at minimum | Minimum misprediction rate |
| --- | --- | --- |
| gcc | m = 18 | 11.17% |
| jpeg | m = 13 | 7.59% |
| perl | m = 14 | 8.82% |

Each benchmark bottoms out and then flatlines. Past that point every static branch already owns a private counter, so there is no aliasing left to remove and extra capacity buys nothing — the only remaining path to accuracy is a better *algorithm*, not a bigger table.

The differing saturation points are themselves a measurement. gcc needs 32x more entries than jpeg before it flattens, which says gcc has far more distinct static branch sites. The predictor's capacity requirement is a proxy for the program's control-flow complexity.

### Global history: harmful, then essential

Sweeping history length `n` from 0 to `m` at every table size, on gcc:

| `m` | Best `n` | Gshare | Bimodal (n=0) |
| --- | --- | --- | --- |
| 7 | 0 | 26.65% | 26.65% |
| 10 | 0 | 15.67% | 15.67% |
| 11 | 1 | 13.64% | 13.65% |
| 13 | 7 | 10.56% | 11.72% |
| 16 | 9 | 7.49% | 11.21% |
| 18 | 10 | 6.73% | 11.17% |
| 20 | 11 | 6.37% | 11.17% |

At `m` of 10 or below, the optimal history length is **zero** — gshare's best move is to become bimodal. Below that threshold global history actively hurts.

The reason is that history *multiplies* the counters a single branch needs. Without history one static branch uses one counter. With `n` bits of history, that branch's dynamic instances scatter across up to 2^n counters, each learning a more specialized pattern. That specialization is only affordable if the table has counters to spare. In a small table it just causes aliasing, and unrelated branches destroy each other's state.

Once the table is large enough the relationship inverts completely. Bimodal is stuck at 11.17% no matter how much capacity you give it, while gshare keeps improving to **6.37%** — a 43% relative reduction. Bimodal cannot exploit abundant counters; gshare can.

This is the most useful result in the project, because it is a genuine non-monotonicity: the *same* feature is a liability at one budget and the decisive advantage at another.

## Dynamic instruction scheduling

A nine-stage superscalar out-of-order pipeline — fetch, decode, rename, register read, dispatch, issue, execute, writeback, retire — with the three structures that make out-of-order execution possible:

* **Reorder buffer**, tracking in-flight instructions so results become architecturally visible in program order
* **Rename map table** over 67 architectural registers, mapping architectural names to reorder-buffer tags and removing false dependencies
* **Issue queue**, holding instructions until operands are ready and selecting which to execute each cycle

Execution latencies vary by functional-unit type, so the model reproduces real timing behaviour rather than assuming uniform single-cycle execution.

### How large an issue queue actually needs to be

Holding the reorder buffer at 512 entries so it could not be the bottleneck, I swept issue queue size from 8 to 256 against pipeline widths of 1, 2, 4 and 8, then found the smallest queue reaching within 6% of the IPC achieved at a 256-entry queue:

| Width | gcc | perl |
| --- | --- | --- |
| 1 | 8 | 8 |
| 2 | 16 | 16 |
| 4 | 32 | 64 |
| 8 | 64 | 128 |

The requirement roughly doubles with width, which follows directly from what the queue is for. A wider machine needs more *independent* instructions every cycle, and the only way to find them is to look further down the dynamic instruction stream — which means holding more of it.

The divergence at width 8 is the interesting part. perl needs twice the queue gcc does for the same relative performance, which means perl's instruction stream carries more dependent work per unit of window: either denser data dependencies, or more long-latency operations occupying slots while their consumers wait. Either way, you have to look twice as far to find eight independent things to do.

The practical lesson is that issue queue size is not an independent design knob. It is determined by width and by workload behaviour, and sizing it without characterizing both leaves either performance or area on the floor.

## Notes for revisiting this

### 2-bit saturating counters

Two bits give four states — strongly and weakly taken, weakly and strongly not-taken — and a prediction flips only after two consecutive contradictions. That hysteresis is the entire point: a single anomalous outcome, such as the final iteration of a loop, should not reverse a well-established prediction. A 1-bit predictor mispredicts a loop branch twice per loop; a 2-bit predictor mispredicts once.

### How gshare indexes

The PC is XORed with the global history register, so the same static branch maps to different counters depending on the recent path. This captures correlation — branches whose outcomes depend on earlier branches, as in `if (x) ... if (x && y)`.

### Aliasing

Two unrelated branches mapping to the same counter and corrupting each other's state. Bimodal aliases when there are more static branches than entries. Gshare aliases far more readily, because each branch consumes many counters.

### Why the hybrid predictor exists

It encodes an admission that no single predictor is best everywhere. Biased branches want bimodal; correlated branches want gshare. Rather than choosing globally, the chooser learns per branch which component has been more reliable, and the chooser's own counter hysteresis stops it flip-flopping after a single mistake.

### Register renaming

The rename map table maps architectural registers to reorder-buffer entries, eliminating write-after-read and write-after-write hazards — false dependencies that exist only because the ISA has a finite number of register names. Only true read-after-write dependencies survive renaming, and those are the ones value prediction attacks.

### Why the reorder buffer is separate from the issue queue

They solve different problems. The issue queue holds instructions *waiting to execute* and can release them the moment they issue. The reorder buffer holds every instruction *in flight* until it retires, because in-order commit and precise exceptions require it. An instruction leaves the issue queue early and the reorder buffer late, so the two are sized independently.

### Questions worth being ready for

**Why hold the reorder buffer at 512 while sweeping the issue queue?** To isolate one variable. If both are near their limits you cannot tell which is binding.

**Why a 6% threshold?** It defines "close enough to the performance of an unlimited queue" so the answer is the smallest structure that is not the bottleneck, rather than the largest that still helps.

**Why does gshare lose at small table sizes?** Because history multiplies counter demand. Same reasoning as aliasing above.

### What these projects do not cover

No TAGE or perceptron predictors, no branch target buffer or return address stack modelling, no memory dependence prediction, no cache or memory hierarchy interaction in the scheduling model, and no power or area accounting.

## Context

Built for ECE 563 (Microprocessor Architecture) at NC State University. The cache and memory hierarchy work from the same course is written up separately in the [cache hierarchy simulator](/projects/cache-hierarchy-simulator/) project.
