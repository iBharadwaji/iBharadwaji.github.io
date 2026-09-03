---
layout: page
title: "Configurable Cache and Memory Hierarchy Simulator"
permalink: /projects/cache-hierarchy-simulator/
---

# Configurable cache and memory hierarchy simulator

A trace-driven cache simulator in C++ supporting a configurable two-level hierarchy with LRU replacement, a write-back write-allocate policy, and a stream-buffer prefetcher. I used it to run **112 simulations** of design-space exploration, measuring miss rate and average access time (AAT) across capacity, associativity, block size, and prefetch configuration.

The interesting part of this project was not writing the cache. It was discovering how often the intuitive answer is wrong.

## What the simulator models

| Feature | Detail |
| --- | --- |
| **Hierarchy** | L1 with an optional L2; setting `L2_SIZE = 0` gives an L1-only system |
| **Geometry** | Block size, capacity and associativity configurable per level, from direct-mapped to fully-associative |
| **Replacement** | LRU, using per-set recency counters |
| **Write policy** | Write-back with write-allocate; dirty bits tracked, writebacks counted on eviction |
| **Prefetcher** | `PREF_N` stream buffers of depth `PREF_M`, attached to the last cache level before memory, with LRU across buffers |
| **Timing** | Per-level hit time, miss penalty and miss rate combined into average access time |

```text
./sim <BLOCKSIZE> <L1_SIZE> <L1_ASSOC> <L2_SIZE> <L2_ASSOC> <PREF_N> <PREF_M> <trace>
```

## Capacity and associativity

Sweeping L1 from 1KB to 1MB across direct-mapped, 2/4/8-way and fully-associative, with 32B blocks:

| Associativity | Miss rate at 1KB | Miss rate at 32KB | Converges to |
| --- | --- | --- | --- |
| Direct-mapped | 19.35% | 3.77% | 2.58% |
| 2-way | 15.60% | 2.88% | 2.58% |
| 4-way | 14.27% | 2.64% | 2.58% |
| 8-way | 13.63% | 2.62% | 2.58% |
| Fully-associative | 13.70% | 2.62% | 2.58% |

Every curve converges to **2.58%**. That number is the compulsory miss rate: the cost of touching each block for the first time, which no amount of capacity or associativity can remove. Capacity removes capacity misses, associativity removes conflict misses, and neither touches the floor. Reading that convergence off the graph is how you estimate the compulsory rate without instrumenting for it directly.

## Average access time does not track miss rate

Miss rate falls monotonically as the cache grows. AAT does not.

The best L1-only configuration was **32KB 4-way at 0.8014 ns**, and beyond that point AAT climbs again — a 1MB cache is worse than a 32KB one. Larger caches are slower to access, and once the miss rate has flattened against the compulsory floor there is no benefit left to pay for that hit time with.

This is the single most useful thing the project taught me: optimizing miss rate and optimizing performance are different objectives, and they diverge exactly where the compulsory floor is reached.

## Adding a second level

With a 16KB 8-way L2 behind an L1 varied from 1KB to 8KB, the best configuration became **8KB direct-mapped L1 at 0.7151 ns** — better than any L1-only hierarchy. A small fast L1 backed by an L2 beats a large slow L1, because the L2 absorbs the misses that would otherwise reach memory.

Co-exploring both levels together (L1 1-8KB 4-way against L2 16/32/64KB 8-way) found **L1 2KB 4-way with L2 64KB 8-way at 0.7077 ns**, an **11.7% AAT improvement** over the best single-level design.

## Block size: spatial locality against pollution

Sweeping 16B to 128B blocks at 4-way associativity:

| Cache size | Best block size | Miss rate |
| --- | --- | --- |
| 1KB | 32B | 14.27% |
| 8KB | 64B | 3.86% |
| 32KB | 128B | 1.11% |

Small caches prefer small blocks; large caches prefer large blocks. The 1KB cache shows the tension directly: going from 16B to 32B blocks helps (14.73% to 14.27%) because each miss brings in more useful neighbouring data, but pushing on to 128B hurts badly (20.36%) because a handful of large blocks evict data that was still live. Spatial locality and cache pollution pull in opposite directions, and the crossover point moves with capacity.

## Stream buffers have a threshold, not a slope

The prefetching study used a streaming microbenchmark — `c[i] = a[i] + b[i]` over three 1000-element arrays — with a 1KB direct-mapped L1 and 16B blocks.

| Stream buffers | Depth | L1 miss rate |
| --- | --- | --- |
| Disabled | — | 0.2500 |
| 1 | 4 | 0.2500 |
| 2 | 4 | 0.2500 |
| **3** | **4** | **0.0010** |
| 4 | 4 | 0.0010 |

The 25% baseline is structural. A 16B block holds four 4-byte elements, so one compulsory miss is followed by three hits, forever. Cache size and associativity are completely irrelevant here — the benchmark has no temporal locality at all, so there is nothing for a bigger cache to retain.

The result worth remembering is the cliff. One and two buffers do not help *slightly*; they do not help *at all*. Each stream buffer tracks a single sequential address stream, and the loop walks three arrays concurrently, so with two buffers one array is always evicting another's prefetches. At three buffers every stream gets its own, and the miss rate collapses by a factor of 250. A fourth buffer is never allocated.

Prefetcher sizing is therefore a matching problem against the number of concurrent streams, not a dial you turn up until performance improves.

## Context

Built for ECE 563 (Microprocessor Architecture) at NC State University, taught by Prof. Eric Rotenberg.
