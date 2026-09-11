---
layout: page
title: "$sformatf() for UVM Reporting"
permalink: /uvm/reporting/sformatf/
---

# `$sformatf()`: Building Formatted Strings

`$sformatf()` is a **SystemVerilog system function** that applies a format string to one or more values and returns the resulting string.

> `$sformatf()` builds a string. It does not print or report the string by itself.

## Why it appears in UVM reporting

UVM reporting macros expect their message argument to be a string. A fixed message can be passed directly:

```systemverilog
`uvm_info("DRIVER", "Driver started", UVM_LOW)
```

When a message must include changing values, `$sformatf()` constructs the string:

```systemverilog
`uvm_info(
  "DRIVER",
  $sformatf("Driving addr=0x%0h, data=0x%0h", req.addr, req.data),
  UVM_MEDIUM
)
```

![Flow from a format template and values through sformatf into a UVM report message](/assets/images/uvm/reporting/sformatf-flow.svg)

*`$sformatf()` converts the template and values into one string; the UVM reporting macro then decides whether and where to report it.*

## Syntax

```systemverilog
string result;
result = $sformatf("format string", value1, value2);
```

Example:

```systemverilog
logic [31:0] addr = 32'h1000;
logic [31:0] data = 32'hABCD;
string msg;

msg = $sformatf("addr=0x%0h, data=0x%0h", addr, data);
// msg contains: addr=0x1000, data=0xabcd
```

## Common format specifiers

| Specifier | Formats the value as |
|---|---|
| `%0d` | Decimal |
| `%0h` | Hexadecimal |
| `%0b` | Binary |
| `%s` | String |
| `%t` | Simulation time |
| `%p` | Aggregate value in assignment-pattern form |

The `0` requests the minimum field width needed for the value instead of padding it to a larger default width.

## `$display`, `$sformatf`, and `$sformat`

| Construct | Behavior |
|---|---|
| `$display(...)` | Formats values and immediately prints the result |
| `$sformatf(...)` | Formats values and returns a new string |
| `$sformat(output, ...)` | Writes the formatted result into a supplied string variable |

Use `$sformatf()` when another API, such as a UVM reporting macro, needs the formatted text as an expression.

## Common mistakes

- Calling `$sformatf()` alone and expecting it to print.
- Describing `$sformatf()` as a UVM macro. It belongs to SystemVerilog.
- Using it for a message that contains no variable data.
- Using a format specifier that does not match the value's intended representation.
- Confusing formatting with reporting: `$sformatf()` builds the text; `` `uvm_info `` reports it.

## Interview-ready answer

> `$sformatf()` is a SystemVerilog system function that formats values according to a format string and returns the result as a string. It is commonly used inside UVM reporting macros to construct messages containing dynamic transaction or testbench data. Unlike `$display`, it does not print anything by itself.

## Quick revision

1. What does `$sformatf()` return? **A string.**
2. Does it print by itself? **No.**
3. Is it part of UVM? **No; it is a SystemVerilog system function.**
4. Why is it useful with `` `uvm_info ``? **It constructs a message containing runtime values.**
