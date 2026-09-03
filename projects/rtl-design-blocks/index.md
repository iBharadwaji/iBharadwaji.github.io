---
layout: page
title: "RTL Design Blocks"
permalink: /projects/rtl-design-blocks/
---

# RTL design blocks

A self-directed set of RTL design and timing-optimization problems, each implemented and verified in SystemVerilog. These are the small components that larger designs are assembled from, and working through them individually is a good way to build the vocabulary that CPU and interconnect design assumes you already have.

## CPU and memory structures

| Block | What it does |
| --- | --- |
| **LRU cache replacement** | Least-recently-used victim selection with parameterizable ways |
| **Skid buffer** | Absorbs a cycle of backpressure so a valid/ready interface can stall without losing data or creating a combinational path from ready to valid |
| **Single-cycle arbiter** | Grants one requester per cycle from many, fairly |
| **Priority encoder** | Finds the first set bit in a vector — the primitive underneath free-list allocation and issue-queue selection |
| **Credit-based flow control** | Sender tracks receiver capacity with credits, with deadlock avoidance |
| **Performance counter** | A counter that clears on read, as used for hardware performance monitoring |
| **Atomic counters** | Single-copy atomic increment behaviour |
| **Ordering logic** | Maintains required ordering while allowing as much reordering as correctness permits |

The skid buffer is the one worth singling out. A naive valid/ready interface either drops data when the consumer stalls or creates a combinational path from the consumer's ready back to the producer's valid, which becomes a critical path the moment the two ends are physically far apart. A skid buffer holds one item in reserve so the producer can be told to stop a cycle late, breaking that path. Almost every pipelined interface in a real design has one somewhere.

## Protocol and datapath

AMBA APB event conversion, low-power channel control, parallel-to-serial conversion, FIFO flush, packet decoders, endianness conversion, running-average and compression datapaths.

## Timing optimization

A separate set of problems targeting critical-path reduction rather than function: skid buffers, fast FIFO read paths, wide data muxing, find-first-set, and packet decoders.

These change how you write RTL. A find-first-set that iterates is correct and slow; one built as a balanced tree is correct and fast. Both compile, both simulate identically, and only one closes timing. Working on problems where the specification is "same behaviour, shorter critical path" makes the performance-power-area trade-off concrete in a way that functional exercises never do.
