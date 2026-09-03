---
layout: page
title: "Out-of-Order Instruction Scheduling Simulator"
permalink: /projects/out-of-order-scheduling/
---

# Out-of-order instruction scheduling simulator

A cycle-accurate C++ model of a superscalar out-of-order pipeline, built to answer a design question rather than to demonstrate a design: how large do the scheduling structures actually need to be?

## The pipeline

Nine stages — fetch, decode, rename, register read, dispatch, issue, execute, writeback, retire — with three structures doing the real work.

**Rename map table** over 67 architectural registers, mapping architectural names to reorder-buffer tags. This eliminates write-after-read and write-after-write hazards: false dependencies that exist only because an instruction set has a finite number of register names, not because the data actually depends on anything. Only true read-after-write dependencies survive renaming, and those are the real limit on parallelism.

**Reorder buffer**, tracking every in-flight instruction so results become architecturally visible in program order. This is what allows precise exceptions in a machine that executes out of order: when a fault occurs, everything older has committed and everything younger can be discarded.

**Issue queue**, holding instructions until their operands are ready and selecting up to `WIDTH` of them each cycle.

Fetch, dispatch, issue and retire are all capped at `WIDTH`, and execution latency varies by functional-unit type at one, two and five cycles, so the timing reflects a real machine rather than assuming uniform single-cycle execution.

Every structure is parameterized by size and width, which turns the model into an instrument for design-space exploration rather than a simulation of one particular machine.

## How large an issue queue actually needs to be

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

Eighty-eight configurations in total across the two benchmarks, plus a second sweep varying the reorder buffer from 32 to 512 at each width's optimized queue size.

## Notes for revisiting this

### Why renaming matters more than it first appears

Consider `r1 = r2 + r3` followed by `r2 = r4 + r5`. The second instruction writes a register the first reads, so it looks like it must wait. It does not — the dependency is on the *name* `r2`, not on the data. Give the second instruction a different physical destination and both can execute simultaneously. An ISA with 32 architectural registers creates enormous numbers of these false dependencies, and renaming removes all of them.

### Why the reorder buffer is separate from the issue queue

They solve different problems and have different lifetimes. The issue queue holds instructions *waiting to execute* and releases them the moment they issue. The reorder buffer holds every instruction *in flight* until it retires, because in-order commit and precise exceptions require it. An instruction leaves the queue early and the buffer late, so the two are sized independently — which is exactly why the experiment holds one fixed while sweeping the other.

### Wakeup and select

Each cycle the issue queue must wake instructions whose operands just became available, then select up to `WIDTH` of them to execute. Wakeup broadcasts result tags to every waiting entry; select arbitrates among the ready ones. Both scale badly with queue size, which is the real reason issue queues stay small in practice — it is a timing constraint, not a storage one.

### Why IPC and not cycle count

IPC normalizes for the fact that different configurations execute the same instruction count over different cycle counts, so it compares designs directly. When averaging IPC across benchmarks, the harmonic mean is the correct choice, because IPC is a rate.

### Questions worth being ready for

**Why hold the reorder buffer at 512 while sweeping the issue queue?** To isolate one variable. If both structures are near their limits you cannot tell which one is binding.

**Why a 6% threshold?** It defines "close enough to the performance of an effectively unlimited queue," so the answer is the smallest structure that is *not* the bottleneck, rather than the largest that still helps.

**What limits IPC once the queue is large enough?** Width, true data dependencies, and execution latency. At that point the queue is no longer the constraint and growing it further changes nothing.

**How would you reduce the queue size requirement?** Anything that removes dependencies or hides latency: better branch prediction to keep the window full of useful work, memory dependence prediction, or value prediction to break true dependencies outright.

### What this project does not cover

No cache or memory hierarchy interaction — memory operations use fixed latencies. No branch prediction or misspeculation recovery, no load-store queue or memory dependence prediction, no physical register file modelling (renaming targets reorder-buffer tags directly), and no power or area accounting.

## Context

Built for ECE 563 (Microprocessor Architecture) at NC State University, validated against eight provided reference runs plus held-out mystery runs. Related work from the same course: the [branch predictors](/projects/branch-prediction/) and the [cache hierarchy simulator](/projects/cache-hierarchy-simulator/).
