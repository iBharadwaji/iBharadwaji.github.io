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

## Notes for revisiting this

### The core tension

DRAM delivers eight bits per cycle. A 4x4 convolution needs sixteen input values simultaneously. Everything structural in this design — the line buffer, the ping-pong buffers, the burst controllers — exists to bridge that gap. If asked "why is it built this way," that sentence is the answer.

### Why a line buffer rather than a frame buffer

Storing the whole feature map would need 1024 x 1024 elements of on-chip memory. The convolution window only ever needs four consecutive rows, so buffering four rows costs 4 x 1024 and lets the rest stream. On-chip memory is the expensive resource in an accelerator, and this is the standard trick for reducing it from O(n²) to O(n).

### Why ping-pong buffers

Pooling consumes pairs of adjacent activated outputs, so it cannot start until the second of a pair exists. With a single buffer the convolution pipeline would stall whenever pooling was reading. Two buffers let one be filled while the other is drained, decoupling producer from consumer. It is double-buffering, the same idea as a graphics back buffer.

### Reuse is what makes it efficient

As the window slides one column right, three of its four columns are the same data. As it slides one row down, three of its four rows are already buffered. A design that re-fetched the window each position would need roughly 16x the memory bandwidth. Convolution is attractive in hardware precisely because its data reuse is so high and so regular.

### Why the datapath widens

Two 8-bit signed values multiply into 16 bits; the partial products carry 17 to keep the sign safe. Summing sixteen such products needs more headroom still, and pooling accumulates further, hence 20 bits. Truncating early would be cheaper in area but would discard low-order bits exactly where accumulation is most sensitive to them.

### Why saturate rather than wrap

A wrapped overflow turns a large positive activation into a large negative one — the worst possible error for a network, since it inverts the meaning of a strong response. Clamping to the representable maximum preserves the ordering of activations even when it loses magnitude. Saturating arithmetic is standard in fixed-point DSP for exactly this reason.

### Why ReLU is nearly free in hardware

`max(0, x)` on a signed number is a sign-bit test and a mux. That is one of the underappreciated reasons ReLU displaced sigmoid and tanh in practice: those need lookup tables or polynomial approximation, while ReLU costs almost nothing in gates.

### Questions worth being ready for

**Why 8 partial-product units for 16 MACs?** Each unit computes two products and adds them, which balances logic depth per pipeline stage against unit count. Sixteen separate multipliers would need a deeper adder tree in the same stage.

**Is this a systolic array?** No. A systolic array passes data between neighbouring processing elements in a rhythm. This is a fixed pipelined datapath with a sliding window feeding it — simpler, and appropriate for a single fixed kernel size.

**What limits throughput?** DRAM bandwidth at the input, not the arithmetic. The datapath can consume data faster than the 8-bit interface supplies it.

**How would you scale it?** Parameterize the kernel size, widen the DRAM interface, or process multiple output channels in parallel by replicating the datapath and sharing the line buffer — the buffer is the expensive part and the reuse is already there.

### What this project does not cover

No synthesis, so there are no frequency, area, or utilization numbers; no testbench or golden-model comparison in this repository; a fixed 4x4 kernel rather than a parameterized one; no support for stride, padding, or multiple channels; and no AXI or other standard bus interface — the DRAM interface is custom.

## Context

Built for ASIC and FPGA design coursework at NC State University.
