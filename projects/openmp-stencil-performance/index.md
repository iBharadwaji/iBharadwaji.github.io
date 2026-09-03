---
layout: page
title: "Cache-Aware Domain Decomposition in OpenMP"
permalink: /projects/openmp-stencil-performance/
---

# Cache-aware domain decomposition in OpenMP

A performance study of how domain decomposition interacts with cache locality, using a five-point stencil solver over a 1026 x 1026 grid for 5000 timesteps, parallelized in C with OpenMP.

The question was simple: given the same computation and the same number of threads, how much does it matter *how* you divide the work? The answer turned out to be more than the parallelization itself in some cases.

## Method

Three decompositions of the same grid:

* **Row-major** — each thread takes a contiguous band of rows
* **Column-major** — each thread takes a band of columns
* **2D block** — the thread count is factored into a two-dimensional grid and each thread takes a rectangular tile

Each was benchmarked across 1, 2, 4, 8 and 16 threads on an Intel Xeon Gold 6130, averaging **10 runs per configuration** so the numbers reflect the machine rather than a single noisy sample.

## Results

Mean runtime in seconds:

| Threads | 2D block | Row-major | Column-major |
| --- | --- | --- | --- |
| 1 | 46.49 | 54.87 | 56.87 |
| 2 | 23.71 | 28.19 | 28.74 |
| 4 | 12.61 | 15.23 | 15.29 |
| 8 | 7.22 | 8.53 | 8.43 |
| 16 | **4.10** | 5.22 | 4.90 |
| **Speedup at 16 threads** | **11.34x** | 10.51x | 11.61x |

## What the numbers say

The 2D-blocked version wins at **every** thread count, including a single thread — where parallelization plays no part at all. At one thread it is 15% faster than row-major and 18% faster than column-major, purely because a square tile has a better surface-to-area ratio than a long thin band. A five-point stencil reads its four neighbours, so a compact tile keeps more of those neighbours in cache.

That single-thread result is the finding worth keeping. The blocking optimization is not a parallel optimization at all; it is a cache optimization that happens to also parallelize well.

The speedup column is a good lesson in reading benchmarks carefully. Column-major shows the *highest* speedup at 11.61x, better than 2D blocking's 11.34x — and is still slower in absolute terms at every thread count. Speedup is measured against a decomposition's own single-thread baseline, so a slow baseline flatters the ratio. Column-major scales impressively away from a bad starting point. If you optimize for the speedup number rather than the runtime, you will pick the wrong design.

## Notes for revisiting this

### What a five-point stencil is

Each cell's new value is the average of itself and its four neighbours — north, south, east, west. Repeat over the whole grid, 5000 times. It is the discrete form of a diffusion equation, and it is the canonical memory-bound kernel: two floating-point operations per cell against five memory reads, so performance is set almost entirely by how well the memory accesses hit in cache.

### Why 2D blocking wins on one thread

Geometry. A thread owning a band of rows touches a region whose *perimeter* is large relative to its area; a thread owning a square tile touches one whose perimeter is small. The stencil reads neighbours, so every cell on the boundary of your region needs data from outside it. Compact tiles minimize that boundary, so more neighbour reads land on data already in cache.

Concretely: a 1026-wide row band spans the full grid width, so moving down one row evicts most of what you just loaded. A square tile of the same area is roughly 32 x 32, and its rows are short enough that the previous row is still resident when you need it as a neighbour.

This is the same surface-to-volume argument behind cache blocking in matrix multiply, and it is why the result appears at one thread — it has nothing to do with parallelism.

### Why column-major is worst at one thread

Both C arrays and the loop walk rows contiguously. Splitting by column means each thread strides across memory rather than along it, so it touches a new cache line on nearly every access and wastes most of each line it fetches.

### Speedup is a ratio, and ratios lie

Column-major records the highest speedup at 11.61x while being slower than 2D blocking at every single thread count. Speedup is measured against a decomposition's *own* single-thread time, so the slowest baseline flatters the ratio most. Column-major scales beautifully away from a bad start.

Always ask what the denominator is. If someone reports speedup without absolute runtime, they may be reporting how bad their serial code was.

### Why 16 threads does not give 16x

Three reasons worth naming: the serial fraction of the program bounds the achievable speedup regardless of core count (Amdahl's law); the kernel is memory-bandwidth-bound, so adding cores contends for the same bandwidth; and boundary exchange between adjacent regions grows with thread count. Reaching 11.34x of a theoretical 16x on a memory-bound kernel is a reasonable outcome.

### False sharing, the thing to mention unprompted

When two threads write to distinct variables that happen to share a cache line, the line ping-pongs between cores under the coherence protocol even though there is no real data sharing. With row decomposition the boundary rows of adjacent threads are the risk. Padding regions to cache-line boundaries is the standard fix.

### Why ten runs per configuration

Because a single timing on a shared HPC cluster measures scheduling noise as much as the code. Ten runs makes the comparison between decompositions defensible instead of anecdotal.

### What this project does not cover

No NUMA-aware allocation or first-touch placement, no explicit vectorization, no cache-oblivious or time-skewed blocking, no MPI or distributed memory, and no hardware performance counters to confirm the cache-locality explanation directly — the argument rests on runtime behaviour matching the geometric prediction.

## Context

Built for coursework in parallel computer architecture at NC State University, run on the university's Hazel HPC cluster.
