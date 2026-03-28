<h1 align="center">
<img src="docs_sample/images/vexriscv.png" alt="MkDocs icon" width="170">
<br>SWD Debug support in VexRiscv<br>
</h1>

# RISC-V Debug Support 

## RISC-V ISA

RISC-V is an open standard instruction set architecture based on established RISC (Reduced Instruction Set Computer) principles.

Comparing to ARM and x86, a RISC-V CPU has the following advantages: 

- Free : RISC-V is open-source.
- Simple : RISC-V is much smaller than other commercial ISAs.
- Modular : RISC-V has a small standard base ISA, with multiple standard extensions.
- Stable : Base and first standard extensions are already frozen. There is no need to worry about major updates. 
- Extensibility : Specific functions can be added based on extensions. There are many more extensions are under development. 

---

### RISC-V Minimal Viable Debugger

To create a minimum viable debugger on a RISC-V Processor, you need the following capabilities:

*   **Memory Access**: The ability to **peek and poke** memory. This means reading from and writing to specific memory locations. Alternatively, the ability to push instructions (through the program buffer) can also suffice if direct memory access is not possible or is restricted.

*   **CPU Register Access**: The ability to **read and write CPU registers**. Registers are special storage locations within the CPU that are used for calculations, control, and status information.

*   **CPU Control**: The ability to **control the CPU**. This involves several key actions:
    *   **Halting the CPU**: Stopping the CPU's execution.
    *  **Single-stepping the CPU**: Executing one instruction at a time.
    *   **Resetting the CPU**: Restarting the CPU.
    *   **Setting Breakpoints**: Pausing the CPU when it reaches a certain instruction or memory location.

Here's a simplified diagram representing these core components:

```
     +---------------------+      +-----------------------+
     |    Debugger Host    |      |    Target Device      |
     +---------------------+      +-----------------------+
             |                           |
             |      (Debug Interface)    |
             v                           v
    +-----------------------+      +-----------------------+
    |   Memory Access       |----->| Memory                |
    |  (Read/Write)         |      |                       |
    +-----------------------+      +-----------------------+
             |                           |
             |                           |
    +-----------------------+      +-----------------------+
    | CPU Register Access   |----->| CPU Registers         |
    |  (Read/Write)         |      |                       |
    +-----------------------+      +-----------------------+
             |                           |
             |                           |
    +-----------------------+      +-----------------------+
    |   CPU Control         |----->| CPU (Halt, Step, etc.)|
    | (Halt, Step, Reset)   |      |                       |
    +-----------------------+      +-----------------------+
```

---

### Vexriscv Processor

VexRiscv is an FPGA-friendly CPU core that implements the RISC-V instruction set architecture (ISA). Designed with flexibility and scalability in mind, it caters to a wide range of applications, from compact microcontroller-based systems to more complex multi-core configurations. The project is open-source under the MIT license, promoting community collaboration and adaptation. 

VexRiscv is developed using SpinalHDL, a high-level hardware description language that enhances design modularity and reusability. This approach allows for a plugin-based architecture, enabling users to customize and extend the CPU's capabilities to meet specific project requirements.

---

#### JTAG support in Vexriscv

JTAG stands for **Joint Test Action Group**. JTAG support in VexRiscv is implemented through a combination of hardware and software components that adhere to the JTAG standard (IEEE 1149.1).

The diagram  below describe how a host computer (with GDB) uses a JTAG interface to debug a VexRiscv CPU.

```

+-----------------+
|  GDB (Host PC)  |
+-------+---------+
        |
        v
+-----------------+     +--------------------+
|   OpenOCD       | --> |  JTAG Dongle       |
|(VexRiscv Driver)|     | (Jtag Uart Cable)  |
|   (Host PC)     |     |(TCK, TMS, TDI, TDO)|
+-----------------+     |                    |
                        +--------------------+
                                  |
                                  v
                    +--------------------------------------------------------------+
                    |                     Target                                   |
                    |    +-----------------+    +-------------------+              |
                    |    |  JTAG Bridge    |--->|  VexRiscv Debug   |              |
                    |    |(TAP, IR, DR,    |    |     Plugin        |              |
                    |    |State Machine)   |    |(SystemDebugBus)   |              |
                    |    +-----------------+    +--------+----------+              |
                    |                                | (CPU Control, Registers)    |
                    |                                v                             |
                    |                     +--------------------------+             |
                    |                     |      VexRiscv CPU        |             |
                    |                     +--------------------------+             |
                    +--------------------------------------------------------------+

```

Debugging VexRiscv using JTAG involves connecting a JTAG dongle to the FPGA, and then using OpenOCD with a VexRiscv specific driver, to communicate with the JTAG bridge, which accesses the CPU debug module, all while GDB executes on the host machine.
                                     
##### Host Setup

Terminal 1 on Host :
```                                              
  openocd -f interface/ftdi/olimex-jtag-tiny.cfg -f target/riscv.cfg
```

Terminal 2 on Host :
```
  riscv32-unknown-elf-gdb hello.elf
  (gdb) target remote :3333
  (gdb) load
  (gdb) break main
  (gdb) continue
  (gdb) info registers
  (gdb) next
  (gdb) print message
  (gdb) x/16x 0x80000000

```

##### Target Setup

<h1 align="center">
<img src="docs_sample/images/fpga_target.jpg" alt="MkDocs icon" width="470">
</h1>

---

### SWD support in Vexriscv

SWD stands for **Serial Wire Debug**. The key differences between  JTAG and SWD are :

* SWD uses a two-wire interface (SWCLK and SWDIO) whereas JTAG uses four wires.
* SWD is generally faster than JTAG due to less overhead.

The following conceptual diagram explains the general components and data flow involved in debugging via SWD:

<img src="docs_sample/images/SWD.png" alt="MkDocs icon" width="2070">

In the diagram:
* The Debug Host PC runs debugging software such as GDB and OpenOCD.
* The Debug Transport Module (DTM) converts the USB/JTAG interface to SWD, communicating with the target system through the SWCLK and SWDIO pins.
* The SW-DP (Debug Port) handles the SWD protocol and provides access to the Debug Module(DM).
* The Debug Module(DM) provides access to the CPU's registers, controls the CPU execution, and provides access to system memory via the system bus.
* The VexRiscv CPU and the Memory System are debugged through this interface.


To add support for the RISC-V debug standard over SWD in VexRiscv below steps need to be followed:

#### 1. Debug Module (DM) Implementation:

* RISC-V debug standard DM handles all control operations regarding Halting and Resuming the CPU.

#### 2. Debug Transport Module (DTM) Implementation:

* DTM is used  to handle the SWD protocol. It should provide access to the DM through the SWD interface by acting as a bridge between the SWD protocol and the DMI. 
* DTM needs to manage the communication with the DM, handling address and data transfers.

#### 3. Mapping the Debug Module Interface (DMI) Implementation:

* DMI is a bus used to communicate with the Debug Module, and it is mapped into the memory space.



---

## Breakdown of Tasks

### **1. Implement Standard Debug spec in Vexriscv**
**Goal:** Extend current implementation with spec compliant implementation.

Detailed tasks and milestones planned for implementation can be found at [Standard Debug Spec Milestone](https://saketsinha.de/standard-debug-vexriscv).

---


## **2. SWD Debug Transport Implementation**
**Goal:** Add Serial Wire Debug support alongside JTAG.
**Reference Specs:** [ARM Debug Interface Architecture Specification ADIv6.0](https://developer.arm.com/documentation/ihi0074/latest/)

The following **Architecture diagram**  explains the components and data flow involved in debugging VexRiscv via SWD, showing both existing (implemented) and new (to be built) components with their implementation phases:

```
+===========================+
|     Debug Host PC         |
|   GDB + OpenOCD           |
|   (transport select swd)  |
+=============+=============+
              |
              | USB (CMSIS-DAP / J-Link / ST-Link)
              v
+===========================+
|     Debug Adapter         |
|  USB-to-SWD Converter     |
|  (probe hardware)         |
+=============+=============+
              |
              | SWCLK + SWDIO (2 wires)
              v
+=============================================================+
|                    Target System (FPGA)                     |
|                                                             |
|  +-------------------------------------------------------+  |
|  | SWD Interface  [Phase 2A]                             |  |
|  |                                                       |  |
|  |  SWD Pins ──> SWD Protocol State Machine              |  |
|  |  (SWCLK,      - 8-bit packet request parser           |  |
|  |   SWDIO)      - 3-bit ACK generator (OK/WAIT/FAULT)   |  |
|  |               - Turnaround & parity handling          |  |
|  |               - Line reset detection (50+ clocks)     |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                              v                              |
|  +-------------------------------------------------------+  |
|  | SW-DP (Debug Port)  [Phase 2B]                        |  |
|  |                                                       |  |
|  |  DP Registers:                                        |  |
|  |  +--------+  +-----------+  +--------+                |  |
|  |  | DPIDR  |  | CTRL/STAT |  | SELECT |                |  |
|  |  | (ID)   |  | (sticky   |  | (AP/   |                |  |
|  |  |        |  |  errors)  |  |  bank) |                |  |
|  |  +--------+  +-----------+  +--------+                |  |
|  |  +--------+  +-----------+                            |  |
|  |  | RDBUFF |  |   ABORT   |                            |  |
|  |  | (read  |  | (error    |                            |  |
|  |  |  buf)  |  |  clear)   |                            |  |
|  |  +--------+  +-----------+                            |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                              | AP Access                    |
|                              v                              |
|  +-------------------------------------------------------+  |
|  | DMI Bus Adapter  [Phase 2C]                           |  |
|  |                                                       |  |
|  |  AP addr + data  ──>  DebugBus.cmd (DebugCmd)         |  |
|  |  DebugBus.rsp (DebugRsp)  ──>  RDATA + ACK            |  |
|  |  (cross-clock-domain via ccToggle)                    |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                              | DebugBus                     |
|                              | (DebugCmd / DebugRsp)        |
|                              v                              |
|  +---------------------------+---------------------------+  |
|  | Debug Module (DM)  [EXISTING - DebugModule.scala]     |  |
|  |                                                       |  |
|  |  dmcontrol (0x10)  |  dmstatus (0x11)                 |  |
|  |  abstractcs (0x16) |  command (0x17)                  |  |
|  |  data0-N (0x04+)   |  progbuf0-N (0x20+)              |  |
|  |  sbcs (0x38)       |  sbaddress0/sbdata0              |  |
|  |  hartinfo (0x12)   |  haltsum0 (0x40)                 |  |
|  +-------------+----------------+------------------------+  |
|                |                |                           |
|    DebugHartBus|                | SBA (System Bus Access)   |
|    (halt/resume|                |                           |
|     signals,   |                v                           |
|     dmToHart,  |  +-----------------------------------+     |
|     hartToDm)  |  | Memory System Bus                 |     |
|                |  | (Tilelink / AXI / Wishbone)       |     |
|                |  +-----------------------------------+     |
|                v                                            |
|  +-------------------------------------------------------+  |
|  | VexRiscv CPU  [EXISTING - CsrPlugin.scala]            |  |
|  |                                                       |  |
|  |  Debug CSRs:        Trigger Module:                   |  |
|  |  dcsr (0x7B0)       tselect (0x7A0)                   |  |
|  |  dpc  (0x7B1)       tdata1  (0x7A1) - type 2          |  |
|  |                     tdata2  (0x7A2) - PC match        |  |
|  |  Debug Mode:        tinfo   (0x7A4)                   |  |
|  |  entry/exit,                                          |  |
|  |  step FSM,          Instruction injection,            |  |
|  |  ebreakm/s/u        data CSR (0x7B4)                  |  |
|  +-------------------------------------------------------+  |
|                                                             |
+=============================================================+
```

**Legend:**
- `[NEW - Phase 2x]` = Components to be built for SWD support (Task 2)
- `[EXISTING - filename]` = Already implemented components
- `DebugBus` = Same internal bus interface used by both JTAG DTM and SWD DTM
- The Debug Module and VexRiscv CPU are **transport-agnostic** — they work identically whether accessed via JTAG or SWD


**Key Insight:** 

Phase 2A (SWD Interface) + Phase 2B (SW-DP) together are the **SWD equivalent of the existing JTAG DTM**. They handle the wire protocol and produce `DebugBus` commands — the same `DebugCmd`/`DebugRsp` interface that the JTAG DTM already produces. 

**The Debug Module and VexRiscv CPU need NO changes** — they only see `DebugBus` commands and have no knowledge of whether they came from JTAG or SWD.

**SoC-level change:** Only `DebugModuleFiber.scala` needs a new `withSwdTransport()` method alongside existing `withJtagTap()`, so the SoC builder can choose which transport to instantiate (or both, with a mux at the `DebugBus` level).

```
JTAG Path (EXISTING):                        SWD Path (NEW):
==============================               ==============================

JTAG TAP + dtmcs/dmi                         SWD Interface (Phase 2A)
(DebugTransportModuleJtag.scala)             + SW-DP registers (Phase 2B)
  - JTAG state machine                        - SWD protocol state machine
  - IR: IDCODE(0x01), dtmcs(0x10),            - 8-bit packet parser
    dmi(0x11)                                  - 3-bit ACK generator
  - DMI op/address/data in dmi DR              - DP regs: DPIDR, CTRL/STAT,
  - dmiStat busy/error handling                  SELECT, RDBUFF, ABORT
              |                                          |
              |  both produce the                        |
              |  same interface                          | DMI Bus Adapter
              v                                          | (Phase 2C)
        DebugBus (DebugCmd/DebugRsp)                     v
              |                                    DebugBus (DebugCmd/DebugRsp)
              |                                          |
              +──────────────► same bus ◄────────────────+
                                  |
                                  v
                     Debug Module (DM)  ◄── NO CHANGES NEEDED
                     (DebugModule.scala)    (transport-agnostic)
                                  |
                          DebugHartBus
                                  |
                                  v
                     VexRiscv CPU        ◄── NO CHANGES NEEDED
                     (CsrPlugin.scala)      (transport-agnostic)
```

**Why Phase 2C (DMI Bus Adapter) is needed:** In the JTAG path, the `dmi` shift register directly carries DMI address+data+op, so the translation to `DebugBus` is straightforward. In SWD, access goes through DP registers — the debugger writes `SELECT` to pick an AP address, then issues AP read/write operations. The adapter translates this DP/AP register access model into `DebugBus` read/write commands.

**Suggested File:** `DebugTransportModuleSwd.scala` (alongside existing `DebugTransportModuleJtag.scala`)

### Phase 2A: SWD Protocol State Machine
**Spec Reference:** ADIv6.0 Sec B4.1-B4.2

**Tasks:**
- [ ] Implement 2-wire physical interface: SWCLK (clock output) + SWDIO (bidirectional data with tristate control)
- [ ] Implement packet request parser — 8-bit frame: Start(1), APnDP(1), RnW(1), A[2:3](2), Parity(1), Stop(1), Park(1)
- [ ] Implement ACK response generator — 3-bit: OK(0b100), WAIT(0b010), FAULT(0b001) (ADIv6.0 Table B4-1)
- [ ] Implement turnaround period management — direction change on SWDIO between host-driven and target-driven phases (Sec B4.1.3)
- [ ] Implement WDATA phase — 33-bit host→target: WDATA[0:31] + parity (for write operations after OK ACK)
- [ ] Implement RDATA phase — 33-bit target→host: RDATA[0:31] + parity (for read operations after OK ACK)
- [ ] Implement even parity checker/generator — separate parity on packet request (4 bits: APnDP, RnW, A[2:3]) and data transfer (32 bits) (Sec B4.1.6)
- [ ] Implement LSB-first bit ordering for all data values (Sec B4.1.5)
- [ ] Implement idle cycle insertion — SWDIO LOW between transactions (Sec B4.1.4)
- [ ] Implement line reset detection — 50+ clock cycles with SWDIO HIGH, followed by 2+ idle cycles (Sec B4.3.3)
- [ ] Start with SWD protocol version 1 (point-to-point); version 2 multi-drop support is optional

### Phase 2B: DP Register File
**Spec Reference:** ADIv6.0 Sec B2.2 (DP Reference Information)

**Tasks:**
- [ ] Implement `DPIDR` register (DP read, A[3:2]=0b00) — read-only device ID with manufacturer, version, min/revision fields
- [ ] Implement `CTRL/STAT` register (DP read/write, A[3:2]=0b01, SELECT.DPBANKSEL=0x0) — sticky error flags: STICKYERR, STICKYCMP, STICKYORUN, WDATAERR; power control: CDBGPWRUPREQ/ACK, CSYSPWRUPREQ/ACK; ORUNDETECT enable
- [ ] Implement `SELECT` register (DP write, A[3:2]=0b10) — DPBANKSEL (4 bits) + ADDR (AP address selection)
- [ ] Implement `RDBUFF` register (DP read, A[3:2]=0b11) — read buffer for previous AP read result
- [ ] Implement `ABORT` register (DP write, A[3:2]=0b00) — DAPABORT, STKCMPCLR, STKERRCLR, WDERRCLR, ORUNERRCLR
- [ ] Implement sticky error handling — FAULT response when any sticky flag is set; errors cleared only via ABORT register (Sec B1.2)
- [ ] Implement WAIT response logic — issued when AP/DP access is outstanding or AP read result not yet available (Sec B4.2.3)
- [ ] Implement protocol error state — target stops driving line on parity/stop/park errors; recovers on line reset (Sec B4.2.5)

### Phase 2C: DMI Bus Adapter (SWD→DebugBus Bridge)
**Spec Reference:** Custom bridge layer

**Tasks:**
- [ ] Map AP read/write operations to `DebugBus.cmd` (same `DebugCmd` interface used by JTAG DTM)
- [ ] Implement AP address decoding — translate `SELECT.ADDR` + `A[3:2]` to DMI address for `DebugBus.cmd.address`
- [ ] Implement posted AP reads — first AP read returns UNKNOWN data; result available on next AP read or RDBUFF read (Sec B4.2.2)
- [ ] Wire `DebugBus.rsp` to RDATA output and ACK generation
- [ ] Handle cross-clock-domain between SWD clock (SWCLK) and debug clock domain (reuse `ccToggle` pattern from JTAG DTM)
- [ ] Implement WAIT response when `DebugBus.cmd` is not ready (DM busy)
- [ ] Implement FAULT response when `DebugBus.rsp.error` is set

---

## **3. System Integration into SoC and Toolchain**
**Goal:** Integrate the new debug architecture into the top-level FPGA/System design.

**JTAG Specific Tasks:**
- [ ] Update SoC top-level to expose standard JTAG pins (TCK/TMS/TDI/TDO/TRST)
- [ ] Integrate clock domains for debug (debug clock vs. system clock)
- [ ] Update synthesis/P&R configs
- [ ] Wire DM to hart debug signals (halt request, resume ack, etc.)
- [ ] Wire SBA port to system bus (if implementing System Bus Access)

**SWD Specific Tasks:**
- [ ] Create `DebugTransportModuleSwd` component with `io.swclk` (input) + `io.swdio` (bidirectional) pins
- [ ] Add `withSwdTransport()` method to `DebugModuleFiber` (alongside existing `withJtagTap()`)
- [ ] Provide SWD pinout constraints for FPGA synthesis
- [ ] Wire SWD DTM output to same `DebugBus` as JTAG DTM (select one at SoC level)
- [ ] Add OpenOCD configuration for SWD transport (`transport select swd`, adapter driver config)
- [ ] Add probe-rs configuration for SWD transport
- [ ] Test with SWD-capable debug probes (e.g., CMSIS-DAP, J-Link, ST-Link in SWD mode)
- [ ] (Optional) Implement SWJ-DP — combined SWD/JTAG port with protocol switching sequence (ADIv6.0 Ch B5)

---

## **4. Verification & Test Infrastructure**
**Goal:** Ensure the entire debug pipeline behaves per spec.

**Tasks:**
- [ ] Build simulation testbenches for DTM→DMI→DM→CPU path
- [ ] Create automated JTAG test sequences for DMI operations
- [ ] Test `dmactive` activation/deactivation sequence
- [ ] Test abstract command error handling (all 7 `cmderr` codes)
- [ ] Test EBREAK behavior with `dcsr.ebreakm`/`ebreaks`/`ebreaku` combinations
- [ ] Test single-step across privilege mode transitions
- [ ] Test trigger module: each trigger type, chaining, `dmode` security
- [ ] Test System Bus Access error handling (if implemented)
- [ ] Test authentication mechanism (if implemented)
- [ ] Regression tests for all implemented features

---

## **5. Documentation & Developer Interface**
**Goal:** Make integration/usage clear for future development.

**Tasks:**
- [ ] Write internal architecture documentation (DM/DTM/Trigger block diagrams)
- [ ] Write user-facing “How to debug VexRiscv” guide (using standard OpenOCD)
- [ ] Document which optional spec features are implemented
- [ ] Document WARL field legal values for each register
- [ ] Comment key code sections in new DM/DTM implementation

---
