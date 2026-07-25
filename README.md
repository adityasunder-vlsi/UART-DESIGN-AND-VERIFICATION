# UART Design & Verification (SystemVerilog)

A complete **UART (Universal Asynchronous Receiver/Transmitter)** design paired with a **class-based, layered SystemVerilog testbench** (generator, driver, monitor, scoreboard, environment) built using standard functional verification methodology concepts.

> SystemVerilog UART design & class-based testbench — randomized stimulus, mailbox/event sync, self-checking scoreboard for TX/RX verification.

## 🎯 Project Summary

This project demonstrates end-to-end **ASIC/FPGA functional verification** skills: building a synthesizable RTL design from scratch, then verifying it using an industry-standard, reusable, class-based testbench architecture (the same principles used in UVM, without the UVM library overhead). It covers the full verification loop — **stimulus generation → driving → passive monitoring → self-checking → pass/fail reporting** — backed by waveform-level proof of correctness.

## 🧠 Skills Demonstrated

- RTL design: finite state machines, clock-domain/baud-rate generation, serializer/deserializer logic
- SystemVerilog OOP: classes, constructors, inheritance-free reusable components
- Constrained-random verification: `rand`, `randc`, `randomize()`
- Testbench communication: `mailbox`, `event`, `fork...join_any`
- Layered/modular testbench architecture (generator–driver–monitor–scoreboard–environment)
- Self-checking scoreboards and functional debug via waveform analysis (EPWave/GTKWave)
- Simulation flow: `$dumpfile`, `$dumpvars`, waveform-based verification signoff

## 📡 What is UART?

**UART (Universal Asynchronous Receiver/Transmitter)** is one of the most widely used serial communication protocols in embedded systems and chip design — found in microcontrollers, sensors, GPS/Bluetooth modules, and debug interfaces. It sends data **one bit at a time** over a single wire with **no shared clock** between sender and receiver (hence "asynchronous") — both sides agree on a baud rate in advance and sample data at that fixed rate. Because it's simple, low-pin-count, and universally supported, UART is a standard building block in real SoCs and a common topic in ASIC/FPGA design and verification interviews.

## 📁 Files

| File | Description |
|---|---|
| `design.sv` | UART TX/RX design — `uart_top`, `uarttx`, `uartrx`, `uart_if` interface |
| `tb.sv` | Layered testbench — `transaction`, `generator`, `driver`, `monitor`, `scoreboard`, `environment` |
| `dump.vcd` / waveform screenshot | Simulation waveform for visual verification |

## 🧩 Design Overview

- **`uarttx`** — Serializes an 8-bit input (`dintx`) onto `tx`, driven by an internally generated baud-rate clock (`uclk`), using an `idle → transfer → idle` FSM. Asserts `donetx` when transmission completes.
- **`uartrx`** — Deserializes the incoming `rx` line into an 8-bit `rxdata`/`doutrx` bus using an `idle → start` FSM. Asserts `donerx` when reception completes.
- **`uart_top`** — Instantiates both TX and RX blocks, parameterized by `clk_freq` and `baud_rate`.
- Baud clock (`uclk`) is generated internally in each block by dividing the system clock based on `clk_freq / baud_rate`.

## 🧪 Testbench Architecture

Built as a modular class-based UVM-style environment (without UVM base classes):

- **`transaction`** – Randomized stimulus: operation type (`read`/`write` via `randc`) and 8-bit data (`rand`).
- **`generator`** – Generates and sends `n` randomized transactions to the driver via a mailbox; synchronizes with `drvnext` / `sconext` events.
- **`driver`** – Drives transactions onto the DUT interface (`uart_if`) — performs reset, then either a TX write sequence or an RX read sequence based on `tr.oper`.
- **`monitor`** – Passively samples the interface, capturing both transmitted and received data, and forwards it to the scoreboard.
- **`scoreboard`** – Compares driver-side data vs. monitor-observed data and reports **MATCH / MISMATCH**.
- **`environment`** – Wires all components together (mailboxes + events) and runs `pre_test → test → post_test`.

Data flow:
```
generator → (mailbox) → driver → DUT → monitor → (mailbox) → scoreboard
                     ↑___________event sync___________↑
```

## 📊 Waveform Analysis

*(insert waveform screenshot here, e.g. `![waveform](waveform.png)`)*

The waveform (captured in EPWave) is grouped into three signal sets:

| Group | Belongs to | Key signals |
|---|---|---|
| Top-level interface | `uart_if` | `dintx, donerx, donetx, doutrx, newd, rst, rx, tx` |
| RX internals | `uartrx` | `count, counts, done, rst, rx, rxdata, state, uclk` |
| TX internals | `uarttx` | `count, counts, din, donetx, newd, rst, state, tx` |

**Reset phase (0 – ~10,000,000 ps):** `rst = 1`; internal regs (`donetx`, `din`, `counts`, `state`) show as X (uninitialized) until the first baud clock edge, matching `driver::reset()` holding reset for 5 `uclk` cycles.

**Reset released (~10M ps):** `rx = 1`, `tx = 1` (idle-high, correct for UART), both FSMs sit in `state = 0` (idle).

**RX (read) transaction (~10M – ~40M ps):**
- `rx` pulled low → RX FSM moves `idle → start`.
- `counts` increments `001…008`, driving the 8-bit shift-in loop.
- `rxdata`/`doutrx` visibly rotate (`80, 40, a0, 50, a8, d4, ea, 00`) as each new bit shifts in: `rxdata <= {rx, rxdata[7:1]}`.
- `donerx` pulses once transaction completes → FSM returns to idle; the monitor captures the final `doutrx` value for the scoreboard.

**TX (write) transaction (~40M ps onward):**
- `dintx`/`din` latch the generator's randomized byte (`3b`).
- `newd` pulses high for one `uclk` — driver's "start send" strobe.
- TX FSM: `state 0 (idle) → 2 (transfer)`, matching the enum encoding `transfer = 2'b10`.
- `counts` increments `001…008` while `tx` serializes `din[counts]` bit by bit.
- `donetx` pulses high on completion (~60–65M ps), `tx` returns to idle, `state → 0`.
- Immediately after, `doutrx` starts changing again (`80, c0…`) — the next RX transaction begins.

**What this confirms:**
- ✅ Correct baud-rate clock generation (`uclk` toggling from `count`/`counts` division)
- ✅ TX FSM `idle → transfer → idle` with correct `donetx` timing
- ✅ RX FSM `idle → start → idle` with correct `donerx` timing
- ✅ Serial-to-parallel and parallel-to-serial conversion both verified against driven data
- ✅ Full TX and RX transactions captured end-to-end, ready for scoreboard comparison

## ▶️ How to Run

```bash
# Using any SystemVerilog simulator (e.g. Questa, VCS, Xcelium)
vlog design.sv tb.sv
vsim -c tb -do "run -all"

# Or with a web simulator like EDA Playground
# Design: design.sv | Testbench: tb.sv | Top module: tb
```

## ✅ Actual Simulation Output (Synopsys VCS)

```
[DRV] : RESET DONE
----------------------------------------
[GEN]: Oper : read Din : 113
[DRV]: Data RCVD : 234
[MON] : DATA RCVD RX 234
[SCO] : DRV : 234 MON : 234
DATA MATCHED
----------------------------------------
[GEN]: Oper : write Din : 59
[DRV]: Data Sent : 59
[MON] : DATA SEND on UART TX 59
[SCO] : DRV : 59 MON : 59
DATA MATCHED
----------------------------------------
[GEN]: Oper : read Din : 59
[DRV]: Data RCVD : 86
[MON] : DATA RCVD RX 86
[SCO] : DRV : 86 MON : 86
DATA MATCHED
----------------------------------------
[GEN]: Oper : write Din : 173
[DRV]: Data Sent : 173
[MON] : DATA SEND on UART TX 173
[SCO] : DRV : 173 MON : 173
DATA MATCHED
----------------------------------------
[GEN]: Oper : write Din : 157
[DRV]: Data Sent : 157
[MON] : DATA SEND on UART TX 157
[SCO] : DRV : 157 MON : 157
DATA MATCHED
----------------------------------------
$finish called from file "testbench.sv", line 499.
$finish at simulation time            138850000
V C S   S i m u l a t i o n   R e p o r t
```

Every transaction — regardless of direction (read/RX or write/TX) or random data value — was independently driven, monitored, and cross-checked by the scoreboard with a **100% MATCH rate**, confirming design correctness across randomized stimulus.

## 🛠️ Tech Stack

`SystemVerilog` · Synopsys VCS · Class-based OOP Testbench · Mailboxes · Events · Constrained Randomization · FSM-based RTL Design · EPWave Waveform Debug

## 📌 Future Improvements

- Add functional coverage (`covergroup`) for operation type and data range
- Add concurrent assertions (SVA) for protocol checks (start/stop bit timing, baud alignment)
- Parameterize baud rate mismatch scenarios for negative testing
