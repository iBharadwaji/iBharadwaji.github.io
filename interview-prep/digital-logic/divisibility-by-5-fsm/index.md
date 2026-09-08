---
layout: page
title: "FSM to check divisibility by 5"
permalink: /interview-prep/digital-logic/divisibility-by-5-fsm/
qsection: digital-logic
tags: [fsm, moore-vs-mealy, serial-arithmetic, modular-arithmetic, state-minimisation]
difficulty: medium
asked_by: [apple]
confidence: relearn
last_reviewed: 2026-09-07
related:
  - /interview-prep/digital-logic/divisibility-by-5-fsm-lsb-first/
---

## Question

Design an FSM that checks whether a number is divisible by 5. Bits arrive serially, one per clock, MSB first. Assert an output when the number received so far is divisible by 5. How many states, why, and what are the transitions?

## Key idea

Appending a bit MSB-first turns the value `N` into `2*N + b`, and divisibility only depends on `N % 5`, so a state need hold nothing but the remainder. Five possible remainders means exactly five states, with `r_next = (2*r + b) % 5`.

![State diagram with five states holding the remainder, amber arcs for a one bit and green arcs for a zero bit](/interview-prep/digital-logic/divisibility-by-5-fsm/assets/divisibility-by-5-fsm.svg)

## Answer

This is not a sequence detector. It is arithmetic: each new bit shifts the accumulated value left and lands in the ones place, so `N -> 2*N + b`. Since `N` is unbounded you cannot store it, but you do not need to — only `N % 5` matters, and modular arithmetic distributes over both the multiply and the add.

Let state `Sr` mean "the value so far leaves remainder `r`". Reset to `S0`.

| State | bit 0 | bit 1 |
|---|---|---|
| S0 | S0 | S1 |
| S1 | S2 | S3 |
| S2 | S4 | S0 |
| S3 | S1 | S2 |
| S4 | S3 | S4 |

```verilog
module div5 (input logic clk, rst, inp, output logic oup);
  typedef enum logic [2:0] {S0, S1, S2, S3, S4} state_t;
  state_t state;
  logic   valid;

  always_ff @(posedge clk) begin
    if (rst) begin
      state <= S0;
      valid <= 1'b0;
    end
    else begin
      valid <= 1'b1;
      case (state)
        S0:      state <= inp ? S1 : S0;
        S1:      state <= inp ? S3 : S2;
        S2:      state <= inp ? S0 : S4;
        S3:      state <= inp ? S2 : S1;
        S4:      state <= inp ? S4 : S3;
        default: state <= S0;
      endcase
    end
  end

  assign oup = valid && (state == S0);
endmodule
```

Moore or Mealy is a design choice, not a property of the problem. `oup = (state == S0)` is Moore: registered, glitch-free, but reporting on bits through the *previous* cycle. `oup = (next_state == S0)` is Mealy: valid in the same cycle the bit arrives, matching the wording of the question, at the cost of a combinational path from `inp` to the output. State and transition logic are identical either way.

The arithmetic form generalises and is cheaper than the case statement:

```verilog
  logic [3:0] sum;
  assign sum = {state, 1'b0} + inp;   // 2*r + b, at most 9
  always_ff @(posedge clk)
    if (rst) state <= '0;
    else     state <= (sum >= 5) ? sum - 4'd5 : sum;
```

## Cost

Five states, so 3 state bits. The arithmetic version is one 4-bit adder plus one mux — the compare and the subtract share an adder, since `sum + 4'd11` produces `sum - 5` in its low bits and the `sum >= 5` condition in its carry-out.

Five is the true minimum: all five remainders are behaviourally distinguishable, so no two can be merged.

## Gotchas

- Never write `% 5` in RTL — a general modulo infers a divider. Because `2*r + b <= 9 < 10`, a single conditional subtract of 5 always suffices.
- Always include a `default` in the state `case`. With 3 bits encoding 5 states, values 5 to 7 are unreachable in theory but leave the FSM with no assignment, so it holds its value forever. In simulation `state` is `3'bxxx` before reset, matches nothing, and stays `x` indefinitely.
- `S0` means both "remainder 0" and "nothing received yet", so a bare Moore output asserts out of reset and claims the empty stream is divisible. Qualify it with a `valid` bit.
- Set that valid bit unconditionally (`valid <= 1'b1`), not with `valid <= valid | inp`. The latter tracks "have I seen a one", so a stream of zeros never reports divisible — but zero *is* divisible by 5.
- Verilog syntax traps that bite here: `if (rst) begin ... end` needs its `end` before `else`, and with ANSI-style ports you must not re-declare the ports in the body.
- Verify a trace against the arithmetic, not against your own table. `1010` is 10, so the machine must finish in `S0`.

## Follow-ups

- **Bits arrive LSB first.** Different machine, 20 states — see [the LSB-first variant](/interview-prep/digital-logic/divisibility-by-5-fsm-lsb-first/).
- **Divisible by any N.** Same skeleton with N states and `r_next = (2*r + b) % N`. The single conditional subtract still works because `2*r + b < 2*N`.
- **Output the remainder instead of a flag.** Already there: the state encoding *is* the remainder, so just expose it.
- **Divisible by a power of two.** Degenerate — inspect the low bits, no FSM needed.
- **Divisible by 3.** `r_next = (2*r + b) % 3`, three states.
