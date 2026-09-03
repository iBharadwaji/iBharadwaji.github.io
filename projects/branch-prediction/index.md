---
layout: page
title: "Branch Predictors: Bimodal, Gshare and Hybrid"
permalink: /projects/branch-prediction/
---

# Branch predictors: bimodal, gshare and hybrid

Three branch predictors in C++, characterized by sweeping their entire design space across SPEC traces rather than measured at a single configuration — table sizes from 128 entries to 1M, with global history length swept independently at every size.

| Predictor | Mechanism |
| --- | --- |
| **Bimodal** | 2-bit saturating counters indexed by PC |
| **Gshare** | Counters indexed by PC hashed against an `n`-bit global history register, so one static branch predicts differently depending on the path taken to reach it |
| **Hybrid** | Bimodal and gshare in parallel, with a chooser table of 2-bit counters selecting per branch |

## Where bimodal saturates

| Benchmark | Table size at minimum | Minimum misprediction rate |
| --- | --- | --- |
| gcc | m = 18 | 11.17% |
| jpeg | m = 13 | 7.59% |
| perl | m = 14 | 8.82% |

Each benchmark bottoms out and then flatlines. Past that point every static branch already owns a private counter, so there is no aliasing left to remove and extra capacity buys nothing — the only remaining path to accuracy is a better *algorithm*, not a bigger table.

The differing saturation points are themselves a measurement. gcc needs 32x more entries than jpeg before it flattens, which says gcc has far more distinct static branch sites. The predictor's capacity requirement is a proxy for the program's control-flow complexity.

## Global history: harmful, then essential

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

## Notes for revisiting this

### 2-bit saturating counters

Two bits give four states — strongly and weakly taken, weakly and strongly not-taken — and a prediction flips only after two consecutive contradictions. That hysteresis is the entire point: a single anomalous outcome, such as the final iteration of a loop, should not reverse a well-established prediction. A 1-bit predictor mispredicts a loop branch twice per loop; a 2-bit predictor mispredicts once.

### How gshare indexes

The PC is XORed with the global history register, so the same static branch maps to different counters depending on the recent path. This captures correlation — branches whose outcomes depend on earlier branches, as in `if (x) ... if (x && y)`.

### Aliasing

Two unrelated branches mapping to the same counter and corrupting each other's state. Bimodal aliases when there are more static branches than entries. Gshare aliases far more readily, because each branch consumes many counters.

### Why the hybrid predictor exists

It encodes an admission that no single predictor is best everywhere. Biased branches want bimodal; correlated branches want gshare. Rather than choosing globally, the chooser learns per branch which component has been more reliable, and the chooser's own counter hysteresis stops it flip-flopping after a single mistake.

### Questions worth being ready for

**Why does gshare lose at small table sizes?** Because history multiplies counter demand, and a small table cannot supply it. The specialization becomes aliasing.

**Why does bimodal stop improving?** Once every static branch has a private counter there is no interference left to eliminate. More capacity cannot help an algorithm that has run out of things to learn.

**What would you use in a real design?** Something in the TAGE family, which keeps multiple history lengths simultaneously and picks the longest one that has proven useful for a given branch — effectively the hybrid idea generalized.

### What this project does not cover

No TAGE or perceptron predictors, no branch target buffer or return address stack, no indirect branch prediction, and no power or area accounting.

## Context

Built for ECE 563 (Microprocessor Architecture) at NC State University. Related work from the same course: the [out-of-order scheduling simulator](/projects/out-of-order-scheduling/) and the [cache hierarchy simulator](/projects/cache-hierarchy-simulator/).
