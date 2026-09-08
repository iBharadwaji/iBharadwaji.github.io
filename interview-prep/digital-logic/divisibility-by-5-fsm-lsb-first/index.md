---
layout: page
title: "FSM to check divisibility by 5, LSB first"
permalink: /interview-prep/digital-logic/divisibility-by-5-fsm-lsb-first/
qsection: digital-logic
tags: [fsm, serial-arithmetic, modular-arithmetic, state-explosion]
difficulty: hard
asked_by: [apple]
confidence: relearn
last_reviewed: 2026-09-07
related:
  - /interview-prep/digital-logic/divisibility-by-5-fsm/
---

## Question

Same divisibility-by-5 checker, but bits now arrive **LSB first** — on cycle `k` you receive the bit of weight `2^k`. What replaces the `2*N + b` relation, what must a state hold, and how many states do you need?

## Key idea

LSB-first, the bits you already hold never move; instead each new bit lands one position higher, so `N -> N + b * 2^k`. The remainder alone can no longer predict the next remainder because the bit's weight depends on `k`, so state becomes the pair (remainder, weight) — and since `2^k % 5` has period 4, that is `5 * 4 = 20` states.

![Weight cycling through one, two, four, three, and the twenty-cell state space formed by five remainders times four weights](/interview-prep/digital-logic/divisibility-by-5-fsm-lsb-first/assets/divisibility-by-5-fsm-lsb-first.svg)

## Answer

Compare what each direction needs in order to compute the next remainder:

| Arrival order | Value relation | Needs |
|---|---|---|
| MSB first | `N -> 2*N + b` | remainder only, 5 states |
| LSB first | `N -> N + b * 2^k` | remainder **and** weight, 20 states |

MSB-first the doubling acts on the *accumulated value*, which is computable from the remainder. LSB-first the scaling acts on the *incoming bit*, and the remainder tells you nothing about what the next bit is worth. Concretely: with `r = 0` and an incoming `1`, the new remainder is 1 at cycle 0, 2 at cycle 1, and 4 at cycle 2. Same remainder, same input, three outcomes — so they must be three distinct states.

The weight does not grow without bound, because only `2^k % 5` matters and that repeats:

```
2^0=1, 2^1=2, 2^2=4, 2^3=8->3, 2^4=16->1, ...
```

so `w` cycles `1, 2, 4, 3` with period 4, and each weight is the previous one doubled mod 5. That is one small register, not a lookup table.

```verilog
module div5_lsb (input logic clk, rst, inp, output logic oup);
  logic [2:0] rem;    // N % 5:     0..4
  logic [2:0] wgt;    // 2^k % 5:   1, 2, 4, 3
  logic       valid;
  logic [3:0] acc, dbl;

  assign acc = rem + (inp ? wgt : 3'd0);   // at most 4 + 4 = 8
  assign dbl = {wgt, 1'b0};                // at most 2 * 4 = 8

  always_ff @(posedge clk) begin
    if (rst) begin
      rem <= 3'd0; wgt <= 3'd1; valid <= 1'b0;   // 2^0 % 5 = 1
    end
    else begin
      rem   <= (acc >= 5) ? acc - 4'd5 : acc;
      wgt   <= (dbl >= 5) ? dbl - 4'd5 : dbl;
      valid <= 1'b1;
    end
  end

  assign oup = valid && (rem == 3'd0);
endmodule
```

Trace 10, whose LSB-first bit stream is `0, 1, 0, 1` against weights `1, 2, 4, 3`: the remainder goes `0, 0, 2, 2, (2+3)=5 -> 0`. Ends at 0, and 10 is divisible by 5.

## Cost

Twenty states, encoded as two registers rather than a flat 20-way case — `rem` is 3 bits and `wgt` is 3 bits, a product encoding. All 20 are reachable and pairwise distinguishable (two states with equal remainder but different weight diverge after a single `1`), so 20 is minimal.

Each update is one 4-bit adder plus one mux. `b * w` is not a multiply: `b` is one bit, so it is a mux, or equivalently `{3{inp}} & wgt`.

## Gotchas

- `N + 2*b` is the wrong relation. The weight is not a constant 2, it is `2^k` and it grows every cycle.
- Both updates stay in range for a *single* conditional subtract because `r + b*w <= 8` and `2*w <= 8`, both under `2 * 5 = 10`. That is the general rule: one subtract suffices whenever `x < 2*M`.
- `wgt` must reset to 1, not 0. A zero weight is an absorbing state and the remainder would never change again.
- `{wgt, 1'b0}` for `2*w` states the 4-bit width outright. `wgt << 1` works here only because of Verilog's context-determined width rules for shifts — the concatenation removes the need to reason about it.
- The 4-bit `acc` and `dbl` assigned back into 3-bit registers is safe (post-subtract values are at most 4) but will raise a lint width warning. Slice explicitly if you want it clean.

## Follow-ups

- **Why is MSB first cheaper?** General fact about serial arithmetic, not special to 5: appending at the LSB end is a fixed multiply-by-radix that folds into the state, while LSB-first forces you to carry positional weight.
- **Four bits at a time.** Since `2^4 = 16` is congruent to 1 mod 5, `N % 5` equals the sum of `N`'s hex digits mod 5 — accumulate nibbles with no weight tracking at all. This is the base-16 cousin of summing decimal digits to test divisibility by 9.
- **Divisibility by N, LSB first.** State count is `N` times the multiplicative order of 2 mod `N`. For 5 that order is 4; for 3 it is 2, giving only 6 states.
- **Why does the weight cycle at all?** 2 is invertible mod 5, so repeated doubling walks a cyclic subgroup and must return to 1.
