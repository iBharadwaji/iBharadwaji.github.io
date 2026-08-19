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


With an interface
The same connection can be represented by one interface port.

interface axi_if (input logic clk);
  logic rst_n;
  logic [31:0] awaddr, wdata;
  logic [3:0]  wstrb;
  logic awvalid, wvalid, awready, wready, bvalid, bready;
  logic [1:0] bresp;

  clocking cb @(posedge clk);
    output awaddr, wdata, wstrb, awvalid, wvalid;
    input  awready, wready, bvalid, bready, bresp;
  endclocking

  modport master (clocking cb, output rst_n);
endinterface

module axi_master (axi_if.master m);
  // Access through m.cb.awaddr, m.cb.wvalid, ...
endmodule

Result
The top-level module now connects one axi_if instance instead of dozens of wires.

