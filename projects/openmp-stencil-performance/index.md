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

## Context

Built for coursework in parallel computer architecture at NC State University, run on the university's Hazel HPC cluster.
