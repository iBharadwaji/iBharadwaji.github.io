---
layout: page
title: "Basic gates from a 2:1 mux and from NAND"
permalink: /interview-prep/digital-logic/gates-from-mux-and-nand/
qsection: digital-logic
tags: [shannon-expansion, functional-completeness, de-morgan, mux, nand, lut]
difficulty: medium
asked_by: [apple]
confidence: shaky
last_reviewed: 2026-09-07
---

## Question

Build NOT, AND, OR, XOR, NAND and NOR using only 2:1 multiplexers, and separately using only NAND gates. Why is a 2:1 mux functionally complete, and what is the minimum NAND count for XOR?

## Key idea

A 2:1 mux computes `Y = ~S & D0 | S & D1`, which is exactly Shannon's expansion `F = ~A & F(0,B) | A & F(1,B)`. So drive `S` with `A` and the data inputs are just `F` with `A` forced to 0 and to 1 — read off, never guess. For NAND, massage the expression until its outermost operation is a complemented product, because that shape *is* a NAND.

![A 2 to 1 mux annotated as one Shannon expansion step, beside the minimal four NAND XOR circuit](/interview-prep/digital-logic/gates-from-mux-and-nand/assets/gates-from-mux-and-nand.svg)

## Answer

### From 2:1 muxes

With `Y = S ? D1 : D0`, substitute `A = 0` and `A = 1` into the target function and read off the cofactors:

| Gate | S | D0 = F(0,B) | D1 = F(1,B) | muxes |
|---|---|---|---|---|
| NOT | A | 1 | 0 | 1 |
| AND | A | 0 | B | 1 |
| OR | A | B | 1 | 1 |
| NAND | A | 1 | B' | 2 |
| NOR | A | B' | 0 | 2 |
| XOR | A | B | B' | 2 |

The split is entirely explained by the cofactor column. The cheap three need only `0`, `1`, or the bare wire `B` — all free. The expensive three need `B'`, a complement, which is not a free signal and costs the second mux. Swapping which variable drives the select does not rescue you: expand NAND on `B` instead and the cofactors are `1` and `A'`, still a complement.

Generalising: with `S = A` and each data input drawn from `{0, 1, B}`, one mux realises 9 of the 16 two-variable functions. The other 7 all need a complemented input. If both polarities are already available — common in real silicon, since a flop gives you `Q` and `Q'` — all six collapse to a single mux.

### From NAND gates

| Gate | Construction | NANDs |
|---|---|---|
| NOT | `NAND(A,A)`, since `A·A = A` | 1 |
| AND | `NAND(A,B)` then invert | 2 |
| OR | `NAND(A', B')` = `(A'·B')'` = `A + B` by De Morgan | 3 |
| XOR | see below | 4 |
| NOR | OR then invert | 4 |

The minimal 4-gate XOR, where the trick is that `N1` is a *shared* term used twice:

| Gate | Expression | Simplifies to | Why |
|---|---|---|---|
| N1 | `NAND(A,B)` | `A' + B'` | De Morgan |
| N2 | `NAND(A, N1)` | `(A·B')'` | `A(A'+B') = 0 + A·B'` |
| N3 | `NAND(B, N1)` | `(A'·B)'` | `B(A'+B') = A'B + 0` |
| N4 | `NAND(N2,N3)` | `A·B' + A'·B` | De Morgan on the output |

`N1 = (AB)' = A' + B'` is one signal carrying **both** complements at once. ANDing it with `A` annihilates the `A'` term and leaves `A·B'`; by symmetry `B` leaves `A'B`. The naive sum-of-products route costs 5 because it builds `A'` and `B'` as two separate single-use signals; the fully literal AND-OR translation costs 9.

Four is the proven minimum. By duality, swapping every NAND for a NOR in that table yields **XNOR in 4 NORs**.

### Why a mux is functionally complete

Two arguments, both worth having.

**Reduction.** One mux gives NOT (`D0=1, D1=0`) and one gives AND (`D0=0, D1=B`). `{NOT, AND}` is a complete basis, therefore the mux is complete. Three lines.

**Structural.** One mux is one Shannon expansion step, consuming one variable and leaving two functions of `n-1` variables on its data inputs. Recurse: after `n` levels the data inputs are functions of zero variables, i.e. constants. So any `n`-input function is a binary mux tree of depth `n` with `2^n - 1` muxes and `2^n` constant leaves — which is precisely an FPGA lookup table. A 4-LUT is a 16:1 mux whose data inputs are SRAM cells holding the truth table.

## Cost

Mux counts: NOT 1, AND 1, OR 1, NAND 2, NOR 2, XOR 2. NAND counts: NOT 1, AND 2, OR 3, XOR 4, NOR 4. An `n`-input function needs `2^n - 1` muxes as a tree.

In CMOS a NAND is 4 transistors while an AND is 6 (NAND plus inverter), which is why NAND-based logic is the cheap default.

## Gotchas

- The completeness proof **requires tie-offs to 0 and 1**. A 2:1 mux with all three inputs driven by variables is *not* functionally complete: feed all ones and you get 1, feed all zeros and you get 0, so it is both 0-preserving and 1-preserving, and no circuit of such gates can ever produce NOT. Say "complete over the basis including constants" and you are answering above level.
- In the 4-NAND XOR, `N2` and `N3` are the *complements* of the product terms, not the products. That is the point — the final NAND expands by De Morgan to `N2' + N3'`, which un-inverts and ORs them in one gate. "Isn't N2 inverted?" is the standard follow-up.
- Derive cofactors by substitution, not intuition. Guessing is where people lose XOR and NOR under pressure.
- "Muxes can build any gate, and FPGAs use muxes" restates the claim rather than proving it. The FPGA fact is a *consequence* of Shannon completeness, not evidence for it.

## Follow-ups

- **Build a 4:1 mux from 2:1 muxes.** Three of them, two levels — one Shannon step per select bit.
- **Is NOR functionally complete?** Yes, dual argument: NOR gives NOT and OR.
- **Why are LUTs mux trees?** The recursion above. You change the function by rewriting the SRAM leaves rather than rewiring anything.
- **Cheapest XOR in CMOS?** Not the 4-NAND version — a transmission-gate XOR is 6 transistors.
- **2:1 mux from NANDs?** Four, same count as XOR.
