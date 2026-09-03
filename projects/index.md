---
layout: page
title: Projects
permalink: /projects/
---

# Projects

A collection of personal projects, experiments, and practical work across CPU microarchitecture, hardware verification, and RTL design.

## Computer architecture and performance

* [Out-of-order RISC-V core model with value prediction](/projects/out-of-order-value-prediction/) — a squash-triggered selective filter on top of stride value prediction, lifting harmonic-mean IPC 22.5% across 15 SPEC benchmarks within a 32 KB storage budget
* [Configurable cache and memory hierarchy simulator](/projects/cache-hierarchy-simulator/) — 112 simulations of design-space exploration; L1/L2 co-optimization cut average access time 11.7%, and stream buffers dropped miss rate from 25% to 0.1%
* [Branch predictors and dynamic instruction scheduling](/projects/branch-prediction-and-scheduling/) — gshare cut gcc misprediction from 11.17% to 6.37% where bimodal saturated, and issue-queue sizing characterized against pipeline width
* [Cache-aware domain decomposition in OpenMP](/projects/openmp-stencil-performance/) — 2D blocking beat row-major decomposition by 15% on a single thread, before any parallel speedup

## Verification

* [UVM verification environment for a pipelined LC-3](/projects/lc3-uvm-verification/) — instruction-level architectural checking against a reference model, targeting the pipeline's stall and bypass paths *(in progress)*
* [SystemVerilog verification environment for an I2C master](/projects/i2c-verification-environment/) — 4 covergroups over 18 coverpoints mapped to a 22-item verification plan, closed to 100% register coverage

## RTL design

* [Pipelined CNN accelerator RTL](/projects/cnn-accelerator-rtl/) — five-stage convolution datapath performing 16 multiply-accumulates per cycle, with line buffering and burst DRAM streaming
* [RTL design blocks](/projects/rtl-design-blocks/) — LRU replacement, skid buffers, arbiters, priority encoders, credit-based flow control, and timing-optimization problems
