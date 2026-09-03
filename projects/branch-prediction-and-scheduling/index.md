---
layout: page
title: "Branch Predictors and Dynamic Instruction Scheduling"
permalink: /projects/branch-prediction-and-scheduling/
---

# Branch predictors and dynamic instruction scheduling

Two C++ simulators modelling the structures that let a processor run ahead of its own dependencies: a superscalar out-of-order scheduler, and a set of branch predictors. Both were evaluated across SPEC CPU2006 and CPU2017 traces.

## Dynamic instruction scheduling

A nine-stage superscalar out-of-order pipeline — fetch, decode, rename, register read, dispatch, issue, execute, writeback, retire — with the three structures that make out-of-order execution possible:

* **Reorder buffer**, tracking every in-flight instruction so results become architecturally visible in program order even though they were produced out of order
* **Rename map table** over 67 architectural registers, mapping architectural names to reorder-buffer tags and eliminating false dependencies
* **Issue queue**, holding instructions until their operands are ready and selecting which to execute each cycle

Execution latencies vary by functional unit type, so the model reproduces the timing behaviour of a real machine rather than assuming uniform single-cycle execution.

Every structure is parameterized — reorder buffer size, issue queue size, and pipeline width — which turns the simulator into an instrument for design-space exploration rather than a fixed model of one machine. Widening the pipeline without growing the reorder buffer, for instance, simply moves the bottleneck rather than removing it.

## Branch predictors

Three predictors of increasing sophistication:

| Predictor | Mechanism |
| --- | --- |
| **Bimodal** | A table of 2-bit saturating counters indexed by PC. Cheap, and surprisingly effective on strongly-biased branches |
| **Gshare** | Counters indexed by PC hashed against a global history register, so the same branch can predict differently depending on the path taken to reach it |
| **Hybrid** | Runs bimodal and gshare in parallel and picks between them per branch, using a chooser table of 2-bit saturating counters indexed by branch history |

The hybrid predictor is the interesting one, because it encodes an admission that no single predictor is best everywhere. Some branches are biased and want bimodal; some are correlated with global history and want gshare. Rather than choosing globally, the chooser learns per branch which component has been more reliable, and the counter hysteresis stops it flip-flopping on a single mistake.

## Context

Built for ECE 563 (Microprocessor Architecture) at NC State University. The cache and memory hierarchy work from the same course is written up separately in the [cache hierarchy simulator](/projects/cache-hierarchy-simulator/) project.
