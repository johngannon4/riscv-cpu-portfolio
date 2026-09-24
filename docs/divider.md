# The Pipelined Divider

Turning a 32-iteration combinational division chain into an 8-stage pipeline with
single-cycle throughput, and making it produce RISC-V's exact required answers.

← [Back to overview](../README.md)

---

## The problem

RV32M specifies four division instructions — `div`, `divu`, `rem`, `remu` — and the
processor must implement them without the SystemVerilog `/` or `%` operators (an explicit
constraint of the assignment, and a realistic one: naive inference produces enormous,
slow hardware).

The algorithm is classic restoring shift-subtract long division, one bit of quotient per
iteration:

```
for i in 0..31:
    remainder = (remainder << 1) | msb(dividend)
    if remainder >= divisor:
        remainder -= divisor
        quotient = (quotient << 1) | 1
    else:
        quotient = (quotient << 1)
    dividend <<= 1
```

Each iteration is a 32-bit compare and a conditional 32-bit subtract. Chaining all 32
iterations combinationally produces a correct divider — and a catastrophic critical path.
Thirty-two dependent 32-bit subtractions in series would set the clock period for the
entire processor.

---

## The pipeline

The fix is to cut the iteration chain into registered stages. The chosen split:

**8 stages × 4 iterations per stage.**

```
    ┌────┐   ┌────┐   ┌────┐         ┌────┐
 ──►│ X0 │──►│ X1 │──►│ X2 │── ... ──►│ X7 │──► result
    └────┘   └────┘   └────┘         └────┘
     4 it.    4 it.    4 it.          4 it.
```

- **Latency:** 8 cycles.
- **Throughput:** 1 divide per cycle. A new divide can start every cycle; up to eight can
  be in flight simultaneously.
- **Critical path:** 4 dependent subtractions instead of 32.

The tradeoff is the standard pipelining one — latency for throughput and clock frequency.
Eight cycles of latency is worse than one for an isolated divide, but the clock is fast
enough that everything *else* in the processor runs quicker, and independent divides no
longer serialize.

---

## Signed division

The datapath above is unsigned. RISC-V needs both, and mixing sign handling into the
iteration logic would complicate all eight stages.

The approach keeps the core unsigned:

1. **At the entrance**, convert negative operands to magnitude via two's complement.
2. **Carry the sign decisions alongside the data** through all eight stages — whether the
   quotient needs negating, whether the remainder needs negating, whether the operation is
   signed at all, and whether it wants quotient or remainder.
3. **At the exit**, apply the negations.

The sign rules follow the C-style truncation semantics RISC-V specifies:

- Quotient is negative when the operand signs differ.
- **Remainder takes the sign of the dividend**, not the divisor — the detail most likely
  to be gotten wrong, and one the `riscv-tests` suite checks directly.

---

## Architectural edge cases

RISC-V does not raise exceptions on division faults. It *specifies exact result values*,
which means these cases are correctness requirements, not undefined behavior:

| Case | `div` | `rem` | `divu` | `remu` |
|---|---|---|---|---|
| **Division by zero** | `-1` (all ones) | dividend | all ones | dividend |
| **Overflow** (`INT_MIN / -1`) | `INT_MIN` | `0` | n/a | n/a |

Both are detected and forced at the pipeline boundary rather than falling out of the
iteration logic — the shift-subtract datapath produces garbage for a zero divisor, and
`INT_MIN` has no positive magnitude representable in 32 bits, so the magnitude conversion
can't handle the overflow case on its own.

---

## Integrating with the processor pipeline

An 8-cycle functional unit inside a 1-cycle-per-stage pipeline raises the question of
where the result reappears. The design sends a divide from Execute into the divider and
has it **rejoin the main pipeline at the Memory stage** when it completes:

```
div  x1, x2, x3   F G D X0 X1 X2 X3 X4 X5 X6 X7 M  W
addi x4, x1, 0      F G D  *  *  *  *  *  *  *  X  M  W
                                                 └── ordinary MX bypass
```

The reason this is the right rejoin point: a completed divide sitting in Memory is
indistinguishable from any other instruction in Memory, so it feeds the **existing MX and
WX bypass networks** with no dedicated forwarding paths. The divider needed no special
case in the bypass logic at all.

The cost is that the instruction's full control state — destination register, whether it
writes a register, whether it wants quotient or remainder, its PC and trace status — has
to travel through all eight divider stages alongside the arithmetic, so its identity
survives the trip.

**Independent divides pipeline at full rate:**

```
div x1, x2, x3   F G D X0 X1 X2 X3 X4 X5 X6 X7 M  W
div x4, x5, x6     F G D  X0 X1 X2 X3 X4 X5 X6 X7 M  W
```

**Dependent divides must serialize**, with the younger one stalling until its producer
drains the entire divider pipeline:

```
div x1, x2, x3   F G D X0 X1 X2 X3 X4 X5 X6 X7 M  W
div x4, x5, x1     F G D  *  *  *  *  *  *  *  X0 X1 ... X7 M  W
```

Every stall cycle caused by a divide is tagged with a dedicated `CYCLE_DIV` status in the
trace output, so the cost of division is visible cycle-by-cycle in the cycle-accurate
verification traces rather than hidden inside a generic "stalled."

---

## Cost

The divider is a significant fraction of the processor's logic — eight stages of 32-bit
subtractors and registered intermediate state is not small. Post-place-and-route, the
divider's registers appear in the near-critical timing paths, which is the expected
outcome: it's been pipelined to roughly the same delay budget as the rest of the design,
which is exactly what "correctly balanced" looks like.

---

← [Back to overview](../README.md) · [← AXI-Lite](axi-lite.md) · [Carry-lookahead adder →](adder-cla.md)
