---
layout: page
title: "Pipelined CNN Accelerator RTL"
permalink: /projects/cnn-accelerator-rtl/
---

# Pipelined CNN accelerator RTL

A streaming convolutional neural network accelerator datapath designed in SystemVerilog, performing 16 multiply-accumulate operations per cycle over a 4x4 kernel with activation, pooling and requantization in the same pipeline.

The design constraint that shapes everything here is bandwidth. The accelerator reads feature-map data from DRAM eight bits at a time, and a 4x4 convolution needs sixteen input values simultaneously. Nearly every structural decision follows from bridging that gap.

## Pipeline

Five registered compute stages, each handing off to the next with valid signalling:

| Stage | Function |
| --- | --- |
| **Multiply** | Eight parallel partial-product units, each computing two products, for 16 multiply-accumulates per cycle |
| **Accumulate** | Adder tree reducing the eight partial results to one convolution output |
| **ReLU** | Activation applied to the accumulated result |
| **Average pool** | Pairwise averaging across adjacent activated outputs |
| **Clip** | Saturating requantization back to signed 8-bit |

## Feeding the datapath

The convolution window needs four rows of the feature map available at once, so the design keeps a **four-row sliding line buffer** of 4 x 1024 elements. As the window advances horizontally, the same buffered rows are reused; when it advances vertically, the oldest row is replaced by the next one streaming in from DRAM. This is what makes it a streaming design rather than one that requires the whole feature map in local memory.

Activation output goes into **ping-pong buffers** of 2 x 1024 elements, so pooling can consume one buffer while the convolution pipeline fills the other. Without that, pooling would stall the datapath every time it needed a pair of adjacent results.

Both DRAM sides use burst read and write controllers behind a start/ready handshake — one path loading the 16 kernel weights at startup, the other streaming feature-map rows in and results out.

## Datapath precision

| Signal | Width |
| --- | --- |
| Input pixels and kernel weights | 8-bit signed |
| Partial products | 17-bit signed |
| Pooling accumulator | 20-bit signed |
| Output after clipping | 8-bit signed, saturating |

The widening through the pipeline is deliberate. Multiplying two 8-bit signed values needs 16 bits, summing sixteen of them needs headroom beyond that, and pooling adds more still. Truncating early would be cheaper in area but would lose low-order bits precisely where the accumulation is most sensitive. Saturating rather than wrapping at the output is the other half of that decision: a wrapped overflow turns a large positive activation into a large negative one, which is a far worse error than clamping.

## Context

Built for ASIC and FPGA design coursework at NC State University.
