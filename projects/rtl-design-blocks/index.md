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

## Notes for revisiting this

### Skid buffer

A valid/ready interface has a subtle failure mode. If the producer's `valid` depends combinationally on the consumer's `ready`, you get a long path across whatever distance separates them, and it becomes a critical path as soon as they are far apart on the die. Register the path and you break it, but now the producer learns about backpressure a cycle late and the in-flight item has nowhere to go.

A skid buffer is the one-entry holding register that catches that item. The producer can be told to stop a cycle late without data loss, and `ready` becomes registered rather than combinational. It costs one entry and one cycle of latency to buy full throughput with clean timing — which is why nearly every pipelined interface in a real design has one.

### LRU replacement

True LRU needs recency state per way per set, and the update logic scales badly with associativity. Real caches above four or eight ways use pseudo-LRU tree schemes: one bit per internal node of a binary tree over the ways, so a 16-way set needs 15 bits rather than full ordering. It approximates LRU closely enough while being far cheaper.

### Arbiters

The question is always fairness versus timing. Fixed-priority is a single priority encoder — fast, but starves low-priority requesters. Round-robin is fair but needs rotating state and a wider critical path. Matrix arbiters give true least-recently-granted at O(n²) storage. Which you pick depends on whether starvation or frequency is the binding constraint.

### Priority encoder and find-first-set

Same primitive: given a bit vector, return the index of the first set bit. It appears everywhere in CPU design — free-list allocation, issue-queue selection, interrupt prioritization. The naive form is a linear chain of comparisons with depth O(n). The fast form is a tree with depth O(log n), computing "any bit set" per subtree and using those to steer the result. This is the classic example of why the *structure* of your logic, not the operation, sets the critical path.

### Credit-based flow control

The sender holds a count of buffer slots the receiver has free, decrements on send, and increments when the receiver returns a credit. Unlike a stall signal it needs no round-trip before sending, so it tolerates long or pipelined links — which is why interconnects use it. Deadlock avoidance is the hard part: if two agents each wait for credits the other will only return after making progress, everything stops. Separate virtual channels or guaranteed drain paths break the cycle.

### Read-to-clear performance counters

A counter that resets when read gives you the delta since the last read without a separate clear operation, and without the race between reading and clearing that would otherwise lose events. Standard in hardware performance monitoring units.

### Atomic counters

"Single-copy atomic" means an observer never sees a partially-updated value. In hardware this constrains how the update is implemented — read-modify-write must be indivisible with respect to other accessors, which usually means the operation happens at a point of serialization rather than in the requester.

### Clock domain crossing

Not in this list, but the neighbouring topic and a near-certain interview question. A single bit crossing domains needs a two-flop synchronizer to reduce metastability probability; a multi-bit bus cannot use that approach, because bits may resolve on different cycles and produce a value that never existed. Buses use gray coding (as in an asynchronous FIFO's pointers, where only one bit changes per increment) or a handshake with a data register held stable across the crossing.

### Questions worth being ready for

**Why is a skid buffer better than just registering the output?** Registering alone loses the in-flight item when backpressure arrives late. The skid buffer is specifically the place that item goes.

**What breaks if find-first-set is on the critical path?** Frequency, everywhere it appears. In an issue queue it runs every cycle for every issue slot, so its depth directly bounds the clock.

**Why not use full LRU everywhere?** Storage and update-logic cost grow with associativity, and pseudo-LRU captures nearly all the hit-rate benefit for a fraction of the area.
