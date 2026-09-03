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

## Context

Built for ECE 745 (ASIC Verification) at NC State University.
