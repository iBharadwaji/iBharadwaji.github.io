---
layout: page
title: "UVM Verification Environment for a Pipelined LC-3"
permalink: /projects/lc3-uvm-verification/
---

# UVM verification environment for a pipelined LC-3

*Status: in progress.*

A UVM verification environment for a pipelined LC-3 processor, targeting instruction-level architectural checking against a reference model.

The LC-3 is a small 16-bit teaching architecture — eight general-purpose registers, condition codes, and a compact instruction set — which makes it an unusually good verification target. The instruction set is small enough to cover exhaustively, and the pipelined implementation is complicated enough to be genuinely interesting: it stalls, and it bypasses.

## Device under test

A pipelined LC-3 with the full instruction set across three classes:

* **ALU** — `ADD`, `AND`, `NOT`
* **Control** — `BR` in all condition-code combinations, `JMP`
* **Memory** — `LD`, `LDR`, `LDI`, `LEA`, `ST`, `STR`, `STI`

The pipeline carries stall logic and four bypass paths, and instruction latencies vary from four to seven cycles depending on class. Loads through indirect addressing are the longest, since they take two dependent memory accesses.

## Planned environment

* A layered UVM agent with driver, monitor and sequencer, plus a sequence library covering each instruction class
* A scoreboard performing instruction-by-instruction architectural comparison against a reference model — register file, program counter, and the negative/zero/positive condition codes
* A functional coverage model spanning opcodes, addressing modes, condition-code updates, and the pipeline's stall conditions and bypass paths

## Why the bypass paths are the point

Verifying that `ADD` computes a sum is straightforward. Verifying that it computes the right sum when its operand is still in flight from the instruction two ahead of it is where pipelined processors actually break.

Bypass logic and stall conditions are combinational answers to a timing question, and they interact with every instruction class differently — a value forwarded from the execute stage is not the same as one forwarded from memory, and an indirect load stalls where a register-to-register operation does not. Coverage that only tracks opcodes will report itself complete while leaving the interesting cases untested. That is why the coverage model targets the hazard conditions directly rather than treating instruction coverage as a proxy for them.

## Context

Coursework at NC State University, building on ECE 745 and ECE 748 verification material. This page will be updated with coverage results and findings once the environment is complete.
