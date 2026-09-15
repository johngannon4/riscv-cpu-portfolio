# AXI4-Lite Memory Integration

Replacing a magic single-cycle memory with a real latency-insensitive bus — the hardest
and most instructive part of the project.

← [Back to overview](../README.md)

---

## What changed, and why it's hard

Up through the five-stage design, memory was a convenient fiction: a read issued and
returned within a single cycle, so fetch could be hidden inside a stage and loads could
complete in Memory. Real systems don't work that way. Memory sits behind a bus, responses
take time to come back, and the bus does not promise *when*.

**AXI4-Lite** is the simplified variant of ARM's AMBA AXI protocol: five independent
channels (read address `AR`, read data `R`, write address `AW`, write data `W`, write
response `B`), each a `VALID`/`READY` handshake. It is *latency-insensitive* by design —
either side can stall the other indefinitely, and correctness never depends on timing.

The core tension of this assignment is that **a pipeline is not latency-insensitive.** It
needs an instruction every cycle to avoid bubbles. So the job isn't implementing the
protocol — it's making a design with hard throughput requirements sit correctly on top of
an interface that makes no timing promises, in every combination of stall, flush, and
in-flight request.

The processor acts as the AXI **manager** on two interfaces: a read-only port for
instruction fetch, and a read/write port for data. The subordinate side (a memory with
skid buffers, capable of one transaction per cycle in steady state) was provided.

---

## Instruction fetch: the `G` stage

AXI transactions happen on clock edges. A read request goes out on one rising edge; the
data comes back on the next. That's one cycle of latency that must live *somewhere*, and
it can't hide inside Decode anymore.

The answer is a new pipeline stage, **G** ("Going to instruction memory"), between Fetch
and Decode:

```
        cycle:   0     1     2     3     4     5
  instruction:   F     G     D     X     M     W
       ARADDR:  ─PC───
      ARVALID:  ──▀▀──
        RDATA:  ──────insn──
       RVALID:  ────────▀▀──
```

Fetch holds the PC and drives `ARVALID`/`ARADDR`. G is where `RDATA` arrives. Decode sees
a real instruction for the first time.

A consequence worth noting: **the Fetch stage can no longer be disassembled.** There are
no instruction bits in `F`, only an address. Every debugging view of the front end had to
move a stage later.

---

## The four hard problems

### 1. Sustaining one instruction per cycle

In steady state the fetch interface must issue a request and accept a response every
cycle. The subordinate's skid buffers make one-transaction-per-cycle possible, but the
manager side has to actually drive the handshakes to keep the pipe full:

```
        cycle:   0     1     2     3     4
        insn0:   F     G     D     X     M
        insn1:         F     G     D     X
        insn2:               F     G     D
       ARADDR:  ─PC0──PC1──PC2─
      ARVALID:  ──▀▀▀▀▀▀▀▀▀▀▀▀──
        RDATA:  ──────i0───i1───i2──
```

Any cycle where `ARVALID` drops without cause is a bubble in the pipeline — a directly
measurable performance loss.

### 2. Backpressure without losing or duplicating instructions

This is where most of the subtlety lives. When the pipeline stalls — say a load-use
hazard in Decode — a fetch is *already in flight*. Two things have to happen in the same
cycle, and getting either wrong corrupts execution:

- **Lower `RREADY`.** The response has arrived but the pipeline can't accept it. Dropping
  `RREADY` makes the memory hold `RDATA` stable for an extra cycle rather than dropping
  the instruction on the floor.
- **Lower `ARVALID`.** The PC in Fetch isn't advancing this cycle. If `ARVALID` stays
  high, the same address is requested *twice*, and the pipeline receives a duplicated
  instruction — a silent correctness bug that no simple test catches.

```
        cycle:   0     1     2     3     4     5
        insn0:   F     G     D     D     X     M      ← stalls in D
        insn1:         F     G     G     D     X      ← held in G
        insn2:               F     F     G     D      ← held in F
      ARVALID:  ──▀▀▀▀▀▀▀▀▀──___──▀▀▀▀──            ← dropped during stall
       RVALID:  ──────▀▀▀▀▀▀▀▀▀▀▀▀▀──
       RREADY:  ──▀▀▀▀▀▀▀▀▀──___──▀▀▀▀──            ← backpressure to memory
```

The two signals move together, one cycle apart in effect, for different reasons. That
asymmetry is the kind of thing that only becomes clear at the waveform level.

### 3. Matching responses to requests

**AXI-Lite provides no tag associating a read address with its read data.** There is no
transaction ID in the Lite subset. Responses return in FIFO order, and that's the only
guarantee.

The manager therefore has to track its own in-flight requests: which PC was requested,
how many are outstanding, and which returning `RDATA` belongs to which stage. This is
bookkeeping the single-cycle memory made unnecessary and that the bus quietly makes the
manager's problem.

### 4. Wrong-path responses

A branch resolves in Execute and flushes `F`, `G`, and `D`. But a request for a wrong-path
instruction may already be on the bus — issued a cycle or two ago, response still coming.

That response **cannot simply be ignored**. AXI handshakes must complete; an unconsumed
response leaves the channel wedged. The design consumes and discards the in-flight
wrong-path response while suppressing the issue of any *new* wrong-path request, then
redirects the PC. Flush logic and bus logic are not independent — they have to agree
cycle-by-cycle about what's in flight.

---

## Data memory: simpler, but it restructures the pipeline

The data interface turned out to be much easier, because there are no stalls to contend
with — but it forced code to move between stages:

```
        cycle:   0     1     2     3     4     5
           lw:   F     G     D     X     M     W
       ARADDR:  ─────────────addr──
        RDATA:  ───────────────────data──
                                     └── registered at end of M, used in W
```

- **Address and store data issue at the end of Execute**, not in Memory.
- **The transaction occupies Memory.**
- **The response is registered at the end of Memory** and consumed in Writeback.

Stores use `AW` + `W` with a byte-enable strobe (`WSTRB`) to handle `sb`/`sh`/`sw`
sub-word writes; loads perform the corresponding sub-word extraction with sign or zero
extension per the RISC-V load encodings. Misaligned access is detected and handled.

### The WM bypass disappeared

The most satisfying consequence of the restructuring: in the five-stage design, a store
whose *data* came from an immediately preceding load needed a **WM bypass** —
Writeback into Memory. Once stores issue their data at the end of Execute, that
dependency no longer exists in the pipeline at all, and the entire bypass path was
**deleted**.

MX and WX remain essential. But WM was eliminated not by optimizing it, but by moving
work to a stage where the hazard stopped existing. Restructuring beat forwarding.

---

## What it cost

| | 5-stage, direct memory | 6-stage, AXI4-Lite |
|---|---|---|
| Logic cells | 13,097 | 13,184 |
| F<sub>max</sub> | 17.56 MHz | 17.19 MHz |
| Taken-branch penalty | 2 cycles | 3 cycles |

About **90 additional logic cells** — under 1% — and roughly 2% of frequency, for a full
latency-insensitive standard bus interface. The dominant cost isn't area or timing at all;
it's the extra cycle of branch penalty from the deeper front end, and the considerable
control complexity of keeping stalls, flushes, and in-flight transactions coherent.

The critical path stayed where it was in the five-stage design — out of the Memory-stage
pipeline register through result-selection logic — so the bus never became the limiter.

---

## What this taught

The protocol is a day's reading. The engineering is in the interaction between a
latency-insensitive interface and a pipeline that has opinions about latency, and the
failure modes are almost all *silent*: a duplicated instruction, a dropped response, a
wrong-path transaction left half-complete. None of them announce themselves. They show up
thousands of cycles later as a wrong value in a register, which is precisely why the
cycle-accurate trace diffing described in [pipeline.md](pipeline.md) mattered so much here.

---

← [Back to overview](../README.md) · [← Pipeline](pipeline.md) · [Divider →](divider.md)
