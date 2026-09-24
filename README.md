# RV32IM Processor on FPGA

A 32-bit RISC-V processor written from scratch in SystemVerilog, built up over a semester
from a ripple-carry adder to a six-stage pipelined core that fetches instructions and
accesses data over an **AXI4-Lite** bus, synthesized to a Lattice ECP5 FPGA and running
real compiled C and Rust programs.

> **Why this repo has no RTL.** The processor was built for a university course
> (Penn CIS 4710/5710, Computer Organization & Design) that reuses its assignments
> across semesters. Publishing the source would hand future students a solution set, so
> this repo documents the design, the engineering decisions, and the measured results
> instead.

---

## At a glance

| | |
|---|---|
| **ISA** | RV32IM (base integer + multiply/divide), user-level |
| **Microarchitecture** | 6-stage in-order pipeline: `F → G → D → X → M → W` |
| **Memory interface** | AXI4-Lite, separate read-only instruction port and read/write data port |
| **Language** | SystemVerilog |
| **Target** | Lattice ECP5 (ULX3S board), Yosys + nextpnr open-source flow |
| **Verification** | cocotb + pytest, official `riscv-tests`, Dhrystone, cycle-accurate trace diffing |
| **Result** | Full marks on both pipelined milestones; all functional and cycle-level tests passing |

---

## Measured results

Post-place-and-route numbers from nextpnr for the ECP5-85F, for the final AXI-Lite design
and the preceding directly-connected-memory design:

| Metric | 5-stage (direct memory) | 6-stage (AXI4-Lite) |
|---|---|---|
| Logic cells (`TRELLIS_COMB`) | 13,097 / 83,640 (15%) | 13,184 / 83,640 (15%) |
| Block RAM (`DP16KD`) | 2 / 208 | 2 / 208 |
| DSP (`MULT18X18D`) | 4 / 156 | 4 / 156 |
| Achieved F<sub>max</sub> | 17.56 MHz | 17.19 MHz |
| Timing constraint | 12.50 MHz — met | 15.24 MHz — met |
| Autograded score | 77 / 77 | 83 / 83 |

The interesting result is the **near-zero area cost of the AXI-Lite migration**: adding a
full latency-insensitive bus interface, an extra pipeline stage, and the request/response
tracking that goes with it cost about 90 logic cells — under 1%. The critical path in both
designs runs out of the Memory-stage pipeline register through result-selection logic, not
through the bus, so the bus interface did not become the frequency limiter.

---

## Pipeline

```
        ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  PC ──►│ F │──►│ G │──►│ D │──►│ X │──►│ M │──►│ W │──► regfile
        └─┬─┘   └─┬─┘   └─┬─┘   └─┬─┘   └─┬─┘   └─┬─┘
          │       │       │       │       │       │
       AR req  R resp   decode   ALU/   AXI-L    result
       to imem from     +regfile branch  data    select
                imem     read   resolve  access  +commit
                                    │
                                    ├──► 8-stage pipelined divider
                                    └──► DSP multiplier
```

`G` ("Going to instruction memory") is the stage that makes AXI-Lite work at full
throughput: the read request goes out on one clock edge and the response arrives on the
next, so the fetch latency needs a pipeline stage of its own rather than being hidden
inside Decode.

**Hazard handling**

- **MX / WX / WD bypasses** cover the common register dependencies, forwarding from
  Memory and Writeback into Execute and from Writeback into Decode within the same cycle.
- **Load-use hazards** stall in Decode for one cycle, since the loaded value is not
  available until the end of Memory.
- **Branches** resolve in Execute. With the `G` stage in the pipeline, a taken branch
  flushes three instructions (`F`, `G`, `D`) instead of two — a real, measurable cost
  paid for the decoupled memory interface.
- **Divides** occupy an 8-stage divider pipeline and rejoin the main pipeline at Memory,
  which keeps them bypassable like any other instruction. Independent divides issue
  back-to-back at one per cycle; a dependent divide stalls until its producer drains.

Deep dive: **[docs/pipeline.md](docs/pipeline.md)**

---

## AXI4-Lite memory interface

The final design talks to memory the way a real SoC block does — through a
latency-insensitive AXI4-Lite interface with independent read-address, read-data,
write-address, write-data, and write-response channels, rather than through a
magic single-cycle memory.

The key implementation difficulties were at the seam between a latency-*insensitive* bus and a
pipeline with hard timing requirements:

- **Full throughput under stalls.** The pipeline needs a new instruction every cycle,
  so the fetch interface has to sustain one transaction per cycle in steady state while
  still handling arbitrary stall cycles.
- **Backpressure.** When the pipeline stalls, a fetch is already in flight. Lowering
  `RREADY` holds the response at the memory for an extra cycle, and `ARVALID` has to drop
  in the same cycle — otherwise the stalled PC is fetched twice.
- **Request/response matching.** AXI-Lite provides no tag linking a read address to its
  data. Responses return in order, so the manager side has to track in-flight PCs itself.
- **Wrong-path responses.** A branch resolving in Execute can invalidate a fetch that has
  already been issued. That response still has to be accepted and discarded, not ignored.

Moving stores to issue at the end of Execute also **eliminated the WM bypass** entirely —
a dependency that simply stops existing once the store address and data leave the pipeline
earlier.

Deep dive: **[docs/axi-lite.md](docs/axi-lite.md)**

---

## Pipelined divider

RV32M's `div`, `divu`, `rem`, and `remu` are built on a restoring shift-subtract long
division datapath. A combinational 32-iteration divider works but sets a miserable
critical path, so it was cut into **8 pipeline stages of 4 iterations each**: 8-cycle
latency, but a new divide can start every cycle.

Signed division is handled by converting operands to magnitudes at the pipeline entrance
and carrying the sign-correction decisions alongside the data, so the core datapath stays
unsigned and the negations happen once at the end. RISC-V's specified edge cases —
division by zero, and `INT_MIN / -1` overflow — are detected and forced to their
architecturally required results.

Deep dive: **[docs/divider.md](docs/divider.md)**

---

## Carry-lookahead adder

The processor's addition is done by a hand-built 32-bit carry-lookahead adder instead of
inferred `+`, composed hierarchically: 1-bit generate/propagate cells → 4-bit groups →
8-bit groups → a top-level group that computes the carries into each 8-bit block. This
replaces the O(n) ripple-carry delay chain with a shallow tree.

It was then checked **exhaustively in hardware**: a bitstream runs the adder against all
2³² possible 16-bit operand pairs on the FPGA at 25 MHz, reporting progress on the board's
LEDs. Roughly three minutes of silicon does what no simulation testbench has time to.

Deep dive: **[docs/adder-cla.md](docs/adder-cla.md)**

---

## Verification

Hardware is unforgiving about "mostly working," so the test strategy was layered:

1. **Directed hazard tests** — cocotb tests targeting specific microarchitectural
   scenarios: each bypass path in isolation, load-use in four variants,
   load-to-store-address, back-to-back loads, taken and not-taken branches after a load,
   `x0` bypass suppression, dependent and independent divides.
2. **The official `riscv-tests` suite** — the RISC-V Foundation's own ISA conformance
   tests, run as compiled binaries on the simulated processor.
3. **Dhrystone** — a full compiled C benchmark, the integration test that catches what
   directed tests miss.
4. **Cycle-accurate trace comparison** — the processor emits, every cycle, what is
   retiring in Writeback along with a status code (`CYCLE_DIV`, branch flush, stall, …).
   These traces are diffed against reference traces cycle by cycle, so the design is
   checked not just for computing the right answer but for taking the right number of
   cycles to do it.
5. **Exhaustive hardware checking** — the carry-lookahead adder validated on the FPGA
   against all 2³² 16-bit operand pairs.
6. **Synthesis gating** — a resource check and timing closure run on real ECP5 place-and-route,
   so "it simulates" never got confused with "it's a circuit."

The cycle-accurate traces were the highest-value tool by a wide margin. A functional test
tells you the answer is wrong; a trace diff tells you the exact cycle where the pipeline
first diverged from expected behavior.

---

## Running real software

The processor isn't a simulation artifact — it runs compiled programs on an FPGA:

- **LED demo (C and Rust)** — the first end-to-end bring-up, compiled with a custom linker
  script and loaded into instruction memory as a hex image, driving the board's LEDs.
- **Oscilloscope demo** — a program on the pipelined core driving a waveform out a GPIO
  pin, probed on a scope. Software-generated timing, verified in the analog world.
- **Dhrystone** — the standard benchmark, compiled from C, executing on the final design.
- **An Atari-style game in Rust** (`no_std`, bare metal) on the final AXI-Lite processor,
  with USB HID controller input and HDMI video output.

---

## Development

Built over a semester as a two-person project, in a reproducible Docker/Dev Container
environment with an entirely open-source hardware toolchain:

| Purpose | Tool |
|---|---|
| Simulation | Verilator, cocotb, pytest |
| Synthesis | Yosys |
| Place & route | nextpnr-ecp5 |
| Bitstream | Project Trellis (`ecppack`) |
| Board | ULX3S (Lattice ECP5-85F) |
| Software toolchain | RISC-V GCC, Rust `riscv32im-unknown-none-elf` |

---

## Credits

Built with [@ndavin26](https://github.com/ndavin26) as a two-person team.

Course infrastructure — testbenches, the AXI-Lite memory subordinate (from
[ZipCPU's `wb2axip`](https://github.com/ZipCPU/wb2axip)), the HDMI controller
([Project F](https://github.com/projf/projf-explore)), and the USB HID controller
([nand2mario](https://github.com/nand2mario/usb_hid_host)) — was provided by
[CIS 5710](https://github.com/cis5710/cis5710-homework). The processor itself —
adder, divider, and every datapath from single-cycle through pipelined AXI-Lite —
is our own work.

Documentation drafted with AI assistance; all RTL and design work is our own.
