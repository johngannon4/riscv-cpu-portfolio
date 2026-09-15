# The Carry-Lookahead Adder

A 32-bit adder built as a hierarchy of generate/propagate blocks instead of inferred `+`,
and checked exhaustively on real silicon.

← [Back to overview](../README.md)

---

## Why build an adder by hand

Addition is the single most frequently exercised operation in the processor — every ALU
op, every branch target, every load/store address, every `PC + 4`. Its delay is a
first-order contributor to the clock period.

The obvious implementation is a **ripple-carry adder**: 32 full adders chained, each
waiting for the carry from the bit below. Simple, small, and O(n) in delay — the carry
into bit 31 has to propagate through 31 stages of logic before the result is valid. That
chain was the first thing built in the project, precisely so that its cost would be
felt before anything better was attempted.

A **carry-lookahead adder** breaks the dependency by computing carries in parallel from
generate and propagate signals, trading area for a shallow tree of delay instead of a
long chain.

---

## Generate and propagate

Each bit position produces two signals from its operands:

- **Generate** (`g`) — this position produces a carry-out regardless of carry-in.
  True when both operand bits are 1.
- **Propagate** (`p`) — this position passes a carry-in through to carry-out.
  True when either operand bit is 1.

The value of these signals is that they **compose**. A group of bits has its own group
generate and group propagate, computable from its members without waiting for any carry
to actually arrive:

- A group generates if any member generates and every higher member in the group propagates.
- A group propagates only if *every* member propagates.

Because group signals compose from smaller group signals, the structure can be built as a
tree.

---

## The hierarchy

```
                    ┌──────────────────────────┐
                    │  top-level group logic   │   carries into each 8-bit block
                    └──┬────┬────┬────┬────────┘
            ┌──────────┘    │    │    └──────────┐
        ┌───▼───┐      ┌────▼──┐ ┌──▼────┐   ┌───▼───┐
        │  gp8  │      │  gp8  │ │  gp8  │   │  gp8  │   bits 7:0, 15:8, 23:16, 31:24
        └───┬───┘      └───────┘ └───────┘   └───────┘
      ┌─────┴─────┐
   ┌──▼──┐     ┌──▼──┐
   │ gp4 │     │ gp4 │                        low nibble / high nibble
   └──┬──┘     └─────┘
      │
  ┌───▼───┐
  │  gp1  │  × 32                             per-bit generate/propagate
  └───────┘
```

Four levels, each built from the one below:

| Block | Built from | Produces |
|---|---|---|
| **`gp1`** | raw operand bits | per-bit `g`, `p` |
| **`gp4`** | four `gp1` signals | group `g`/`p` for 4 bits, plus internal carries |
| **`gp8`** | two `gp4` blocks | group `g`/`p` for 8 bits, plus internal carries |
| **top** | four `gp8` group signals | carries into each 8-bit block |

The 32-bit adder instantiates 32 `gp1` cells, four `gp8` blocks covering bits `[7:0]`
through `[31:24]`, and a top-level group that computes the carry into each 8-bit block
from the blocks' group signals. Each `gp8` internally reuses two `gp4` blocks rather than
being written flat — the same composition property applied one level down.

The result: the carry into bit 31 is computed through roughly four levels of group logic
rather than 31 sequential full-adder carries.

---

## Exhaustive verification in hardware

Carry-lookahead bugs are unusually good at hiding from testbenches. An error in a single
group's generate term might only manifest for operand pairs producing one specific carry
pattern across one specific block boundary. Sampling a few million random pairs out of the
2⁶⁴ possible ones can easily miss it — and the processor then computes correct answers for
every test you wrote, and a wrong one somewhere inside Dhrystone a month later.

Directed and randomized cocotb tests in simulation catch the obvious cases. To close the
sampling gap, the adder was also synthesized into a **self-checking bitstream** that runs
on the FPGA and compares it against a reference across **all 2³² possible 16-bit operand
pairs**, reporting progress across eight sections on the board's LEDs — a lit LED per
completed section, a stall at the first failing one.

At 25 MHz the sweep takes just under three minutes. That is the thing hardware can do
that a simulator cannot: four billion exhaustive checks in the time it takes to make
coffee, where the equivalent simulation run would be measured in days.

The lesson generalizes past this component. Coverage is a resource, and where the input
space is small enough to enumerate, enumerate it — don't sample. Nothing else in this
project has a state space small enough for that to be an option, which is exactly why it
was worth doing here.

---

## Where it ends up

The adder is instantiated in the processor's Execute stage and used for ALU addition and
subtraction, with the multi-cycle and pipelined designs both pulling it in as a shared
component. On the final ECP5 place-and-route, the critical path runs through Memory-stage
result-selection logic rather than through the adder — which is the goal. An adder that
is no longer the slowest thing in the design has done its job.

---

← [Back to overview](../README.md) · [← Divider](divider.md)
