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

## Notes for revisiting this

### UVM component roles, in one line each

**Sequence item** — one transaction, as data. **Sequence** — a stream of items describing a scenario. **Sequencer** — arbitrates between sequences competing for the driver. **Driver** — converts items into pin wiggles. **Monitor** — converts pin wiggles back into items, passively. **Agent** — packages sequencer, driver and monitor for one interface. **Scoreboard** — checks observed behaviour against expected. **Environment** — instantiates and connects agents and scoreboards. **Test** — configures the environment and starts sequences.

### Why the monitor must be independent of the driver

The monitor reconstructs transactions from the pins, not from what the driver intended to send. If it shortcut to the driver's data, it would report success whenever the driver was correct, even if the DUT ignored the pins entirely. Passive reconstruction is what makes the check meaningful.

### The UVM phases that matter

`build_phase` constructs components top-down; `connect_phase` wires TLM ports bottom-up; `run_phase` is the only time-consuming phase and is where sequences execute. Objections raised during `run_phase` keep the simulation alive until all are dropped.

### The factory, and why override matters

Components are created through the factory rather than with `new`, so a test can substitute a derived class — an error-injecting driver in place of the normal one, for instance — without editing the environment. That indirection is most of why UVM exists.

### `uvm_config_db`

A hierarchical key-value store for passing configuration and virtual interfaces down the component tree without threading them through constructors. The classic use is handing a virtual interface from the top-level module into a driver several levels deep.

### The three C's of checking

**Scoreboard** compares observed against expected. **Assertions** check temporal properties in-line. **Coverage** measures what you exercised. They answer different questions: is it right, is it right *at every moment*, and did you try hard enough.

### Why the bypass paths are the real target

Verifying that `ADD` computes a sum is trivial. Verifying that it computes the right sum when its operand is still in flight from the instruction two ahead is where pipelined processors actually break.

Bypass logic and stall conditions are combinational answers to a timing question, and they interact with each instruction class differently — a value forwarded from execute is not the same as one forwarded from memory, and an indirect load stalls where a register-to-register operation does not. Coverage that only tracks opcodes will report itself complete while leaving every interesting case untested. That is why the coverage model targets hazard conditions directly rather than treating instruction coverage as a proxy.

### The LC-3 itself

Sixteen-bit words, eight general-purpose registers, and condition codes set from the last value written to a register. Three addressing families: PC-relative (`LD`, `ST`, `LEA`, `BR`), base-plus-offset (`LDR`, `STR`, `JMP`), and indirect (`LDI`, `STI`, which take two dependent memory accesses and are therefore the longest-latency instructions and the most interesting to verify).

### Questions worth being ready for

**Why a reference model rather than expected values per test?** Because per-test expected values only cover what you enumerated. A reference model checks every instruction of every test, including random ones, without additional authoring effort.

**How do you know your coverage model is good?** You do not, from coverage alone — coverage measures the model you wrote, not the space of real bugs. That is why the model is derived from a plan grounded in the design's hazard structure rather than from what happened to be easy to sample.

**What is the difference between this and the I2C environment?** That one verifies a protocol at an interface; this verifies architectural state through a pipeline. The checking target moves from "did the right bytes appear on the wire" to "does the visible machine state match the ISA."

### Status and honesty

This project is not finished. Nothing on this page claims coverage figures, bug counts, or test counts, because none exist yet. The scope above is what is being built, and the page will be updated with results when there are results.

## Context

Coursework at NC State University, building on ECE 745 and ECE 748 verification material.
