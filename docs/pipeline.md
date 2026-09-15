# The Pipeline

How the processor got from one instruction every cycle-and-a-half to a six-stage
in-order pipeline, and what each design decision cost.

← [Back to overview](../README.md)

---

## Evolution

The processor was not designed as a pipeline from the start. It arrived there through
three working processors, each of which had to pass the full test suite before the next
began:

| Design | Structure | Cost per instruction |
|---|---|---|
| **Single-cycle** | Everything in one clock: fetch, decode, execute, memory, writeback | 1 cycle, but a brutal critical path |
| **Multi-cycle** | Single-cycle datapath + multi-cycle divide | 1 cycle, except 9 for divides |
| **5-stage pipeline** | `F D X M W`, direct memory | ~1 cycle, +stalls and flushes |
| **6-stage pipeline** | `F G D X M W`, AXI4-Lite memory | ~1 cycle, +a deeper branch penalty |

Each transition was a real architectural rewrite rather than an extension. The
single-cycle-to-pipelined step in particular meant re-deriving every signal as a
per-stage quantity: an instruction's PC, its decoded control bits, its operand values,
and its status all have to exist independently in five or six places at once.

---

## Stage breakdown

```
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ F │──►│ G │──►│ D │──►│ X │──►│ M │──►│ W │
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
```

**F — Fetch.** Holds the PC and issues the instruction-memory read request. In the
five-stage design this stage also received the instruction; with AXI-Lite it only sends
the address.

**G — Going to instruction memory.** Exists purely to absorb the one-cycle round trip of
the AXI-Lite read: request goes out on one rising edge, `RDATA` comes back on the next.
The instruction bits genuinely do not exist before this stage completes, which means the
Fetch stage can no longer even be disassembled for debugging — there is nothing there but
a PC.

**D — Decode.** Instruction decode, immediate generation and sign-extension for all RISC-V
immediate formats, and register-file reads. Also where load-use stalls are detected and
held.

**X — Execute.** ALU operations through the carry-lookahead adder, branch condition
evaluation and target computation, divide issue into the divider pipeline, DSP-backed
multiplication, and — in the AXI-Lite design — memory address and store-data issue.

**M — Memory.** The cycle during which the AXI-Lite data transaction is in flight. The
response is registered at the end of this stage.

**W — Writeback.** Result selection (ALU, load, multiply, divide, PC+4 for jumps) and the
register-file commit. Also where the per-cycle trace outputs are driven for verification.

Inter-stage state is carried in SystemVerilog `struct packed` pipeline registers, one type
per stage boundary, which keeps a stage's interface explicit rather than scattered across
dozens of individually-named wires.

---

## Bypassing

Without forwarding, nearly every instruction pair with a register dependency would stall
for two or three cycles. Three bypass networks cover the cases:

| Bypass | From | To | Why |
|---|---|---|---|
| **MX** | Memory | Execute | The most common case: a result produced last cycle feeding the very next instruction. |
| **WX** | Writeback | Execute | Covers a two-instruction gap. |
| **WD** | Writeback | Decode | An instruction committing in Writeback in the same cycle a consumer reads the register file. The read would otherwise return the stale value. |

Two details that matter more than they look:

**`x0` must not be bypassed.** Register `x0` is hardwired to zero. An instruction writing
to `x0` produces a value that must be discarded, but a naive tag-match bypass will happily
forward it to a consumer reading `x0` — silently breaking the ISA. The bypass logic
explicitly suppresses `x0` matches, and this has its own test.

**Bypass priority is ordered.** When both MX and WX match the same register, MX wins: the
younger producer holds the correct value. Getting this precedence backwards produces bugs
that only appear in specific instruction spacings, which is exactly the kind of thing
directed tests are for.

---

## Stalls

**Load-use.** A load's data is not available until the end of Memory, so a dependent
instruction immediately behind it cannot execute on schedule. Detection happens in Decode:
if a load sits in Execute and the instruction in Decode reads its destination register,
the pipeline holds Decode for one cycle, lets the load advance, and injects a bubble into
Execute. One cycle is enough — the MX bypass covers the rest.

**Divide.** A divide occupies the divider pipeline for eight cycles. When a dependent
instruction needs the result, the pipeline stalls with a dedicated `CYCLE_DIV` status so
the reason for each stall cycle is visible in the trace output. Independent divides do
*not* stall each other — the divider is pipelined, so back-to-back independent divides
issue at one per cycle.

**Stall priority.** When a load-use stall and a branch flush want the same cycle, load-use
takes priority: a branch in Execute may be comparing stale operands, so resolving it
before the correct data arrives would compute the wrong direction. This ordering was the
source of a real bug, found by a cycle-trace diff rather than a functional failure.

---

## Branches

Branch direction and target are resolved in **Execute**, with the front end fetching
sequentially by default. When a branch is taken, everything younger in the pipeline is on
the wrong path and gets replaced with bubbles.

The penalty is where the AXI-Lite migration shows up as a visible architectural cost:

| Design | Flushed on taken branch | Penalty |
|---|---|---|
| 5-stage | `F`, `D` | 2 cycles |
| 6-stage | `F`, `G`, `D` | 3 cycles |

That extra cycle is the price of a decoupled memory interface. It's the kind of tradeoff
that a branch predictor exists to recover — not in scope here, but the reason the
motivation for one is now obvious from the inside.

The flush also has to coordinate with the bus. A fetch for a wrong-path instruction may
already be in flight when the branch resolves; that response still arrives and must be
consumed and discarded, while no *new* wrong-path request is issued. See
[AXI-Lite integration](axi-lite.md) for how that's handled.

---

## Divide integration

An 8-cycle functional unit inside a 1-cycle-per-stage pipeline is an interesting
structural problem. The approach: a divide leaves Execute into the divider pipeline, and
**rejoins the main pipeline at Memory** when it completes.

```
div  x1, x2, x3   F G D X0 X1 X2 X3 X4 X5 X6 X7 M  W
addi x4, x1, 0      F G D  *  *  *  *  *  *  *  X  M  W
                                                 └── MX bypass from the divide
```

The payoff is that a completed divide looks exactly like any other instruction sitting in
Memory, so it feeds the ordinary MX and WX bypass networks with no special-case forwarding
paths. Control signals for the Memory and Writeback stages ride alongside the operands
through all eight divider stages so the instruction's identity survives the trip.

Independent divides pipeline cleanly:

```
div x1, x2, x3   F G D X0 X1 X2 X3 X4 X5 X6 X7 M  W
div x4, x5, x6     F G D  X0 X1 X2 X3 X4 X5 X6 X7 M  W
```

A dependent divide cannot, and stalls until its producer drains the divider.

---

## Verification approach

The pipeline's tests are organized around the *hazard* rather than the instruction:
each bypass path is exercised in isolation (`testMX1`, `testWX2`, `testWD1`, …),
load-use is tested in four spacings plus interactions with branches, jumps, and store
addresses, and `x0` bypass suppression gets its own case. Roughly forty directed tests sit
underneath the `riscv-tests` suite and Dhrystone.

The cycle-accurate trace diffing is what made pipeline debugging tractable. Every cycle,
the design reports what is retiring in Writeback and why the pipeline is in the state it's
in; that stream is compared against a reference trace. When something breaks, the diff
points at the first cycle of divergence instead of leaving you to work backwards from a
wrong register value a few thousand cycles later.

---

← [Back to overview](../README.md) · [AXI-Lite integration →](axi-lite.md)
