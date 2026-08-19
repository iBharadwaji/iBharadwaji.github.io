---
layout: page
title: Logical Grouping
permalink: /sv/interface/logical-grouping/
---

# Logical grouping

## Why it matters

Reduces port-list length and makes hierarchy easier to read.

## Practical illustration

### Without an interface

A 16-bit AXI-Lite style master can easily need around 30 ports.

```systemverilog
module axi_master (
  input  logic        clk,
  input  logic        rst_n,
  output logic [31:0] awaddr,
  output logic        awvalid,
  input  logic        awready,
  output logic [31:0] wdata,
  output logic [3:0]  wstrb,
  output logic        wvalid,
  input  logic        wready,
  input  logic [1:0]  bresp,
  input  logic        bvalid,
  output logic        bready
  // ... many more signals ...
);
endmodule
