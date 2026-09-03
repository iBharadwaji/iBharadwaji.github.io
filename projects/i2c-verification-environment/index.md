---
layout: page
title: "SystemVerilog Verification Environment for an I2C Master"
permalink: /projects/i2c-verification-environment/
---

# SystemVerilog verification environment for an I2C master

A layered, class-based SystemVerilog verification environment for an OpenCores I2C multi-bus master controller with a Wishbone host interface, driven to functional coverage closure against a written verification plan.

The device under test sits between two protocols at once. A host programs it over Wishbone by writing command and data registers; the controller then drives the I2C bus on the other side. Verifying it means observing both worlds and checking that what happened on the wire matches what the host asked for.

## Environment structure

| Component | Role |
| --- | --- |
| **Wishbone agent** | Driver and monitor for the host-side register interface |
| **I2C agent** | Driver and monitor for the serial bus, including a responding slave model |
| **Generator** | Layered stimulus, organized into seven directed phases plus a randomized burn-in |
| **Predictor** | Decodes host-side register writes into the I2C transactions they should produce |
| **Scoreboard** | Records observed bus-level transfers |
| **Coverage subscriber** | Samples monitored traffic into the functional coverage model |

The environment is built on a class-based framework using configuration lookup for virtual interfaces and analysis-style connections between monitors and subscribers. It is not UVM, though it follows the same architectural pattern — agents wrapping drivers and monitors, stimulus separated from checking, coverage collected passively from monitored traffic rather than from the stimulus generator.

## Coverage model

Four covergroups spanning **18 coverpoints and crosses across 87 bins**, each mapped to an item in a 22-entry verification plan:

* **Register access** — address and read/write direction, crossed, plus data-value ranges on the data port
* **Control and status** — the core enable and interrupt enable bits, and all four combinations of the two
* **Command activity** — every command opcode, the response codes returned by polling, their cross, legal command-to-command transitions, and bus selection
* **Bus transfers** — read against write, slave address ranges, transfer sizes, data values, and the crosses that matter

Writing the plan first and mapping covergroups onto it one-for-one is what made closure measurable. Coverage that is not tied to a plan tells you how much you sampled, not how much you verified.

## Results

| Metric | Result |
| --- | --- |
| Register-access coverage | 100% |
| Control and status coverage | 100% |
| Command coverage | 96.7% |
| DUT toggle coverage | 81% |
| Wishbone transactions driven | 3,214 |

Run under Questa, merging functional and code coverage into a unified database and reporting against the test plan.

## Notes for revisiting this

### Why layering matters

Each layer has one job. The driver knows how to wiggle pins and nothing about intent. The monitor knows how to reconstruct transactions from pins and nothing about correctness. The scoreboard knows about correctness and nothing about pins. Collapsing any two together produces a testbench that cannot be reused when the protocol changes, and one that can hide bugs by checking against its own stimulus rather than against the specification.

### Monitor versus driver, and why coverage samples the monitor

The driver *causes* activity; the monitor *observes* it. Coverage must be sampled from the monitor, because sampling from the driver records what you intended rather than what the design actually did. If the DUT drops a transaction, driver-side coverage still counts it and reports a hole that does not exist.

### Predictor and scoreboard

The predictor models what the design *should* do — decode a host-side register write into the I2C transaction it ought to produce. The scoreboard compares that expectation against what the monitor observed. This is the same reference-model idea as CPU verification, one abstraction level down.

### Why the verification plan comes first

Coverage without a plan measures how much you happened to exercise, not how much you meant to. Mapping four covergroups onto 22 planned items means a coverage report answers "which of my intended checks are unproven" rather than "here is a number." That mapping is why 96.7% command coverage is actionable — you can name the two bins still open.

### Functional coverage versus code coverage

Code coverage (81% toggle here) says which parts of the DUT your stimulus reached. Functional coverage says which *scenarios* you verified. Both are necessary and neither is sufficient: 100% code coverage with no functional coverage means you executed every line without checking whether the behaviour was correct, and 100% functional coverage with poor code coverage means your plan missed whole regions of the design.

### The I2C protocol, quickly

Two open-drain wires, SCL and SDA, with pull-ups. A **start** is SDA falling while SCL is high; a **stop** is SDA rising while SCL is high — the only two times SDA may change while SCL is high, which is what makes them unambiguous framing. Data bytes are eight bits, most-significant first, each followed by an **ACK** from the receiver pulling SDA low. A master reading multiple bytes ACKs each one and **NACKs the last**, which is how it signals "stop sending."

### Why open-drain matters

Devices can only pull the line low, never drive it high; pull-up resistors restore the high state. That is what allows multi-master arbitration without contention — a master that intends to drive high but observes low has lost arbitration and withdraws — and it enables clock stretching, where a slow slave holds SCL low to delay the master.

### Questions worth being ready for

**Why is this not UVM?** It is built on a course framework with the same architecture — agents, configuration lookup, analysis-style connections — but not the UVM base classes. Worth saying plainly rather than letting someone discover it.

**How would you convert it to UVM?** Transactions become `uvm_sequence_item`, the generator's phases become sequences on a sequencer, components extend `uvm_component` with `build_phase` and `connect_phase`, monitor output moves onto `uvm_analysis_port`, and the config lookup becomes `uvm_config_db`. The structure survives; the base classes change.

**Why is command coverage only 96.7%?** Two bins in the command-to-response cross remain unhit. Knowing precisely which holes remain, and why, is the point of mapping to a plan.

**Why randomize at all if directed tests hit the plan?** Directed tests only find bugs you thought of. Randomized burn-in explores orderings and value combinations nobody enumerated, which is where protocol bugs actually live.

### What this project does not cover

No repeated-start sequences, no 10-bit addressing, no clock stretching, no multi-master arbitration, and no fast-mode timing — the DUT supports 100 kHz standard mode here. Assertion-based checking is also absent; the checking is scoreboard-based.

## Context

Built for ECE 745 (ASIC Verification) at NC State University.
