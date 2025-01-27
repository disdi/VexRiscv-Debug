<h1 align="center">
<img src="docs_sample/images/vexriscv.png" alt="MkDocs icon" width="170">
<br>Enhance Debug support in VexRiscv<br>
</h1>



## RISC-V ISA

RISC-V is an open standard instruction set architecture based on established RISC (Reduced Instruction Set Computer) principles.

Comparing to ARM and x86, a RISC-V CPU has the following advantages: 

- Free : RISC-V is open-source.
- Simple : RISC-V is much smaller than other commercial ISAs.
- Modular : RISC-V has a small standard base ISA, with multiple standard extensions.
- Stable : Base and first standard extensions are already frozen. There is no need to worry about major updates. 
- Extensibility : Specific functions can be added based on extensions. There are many more extensions are under development. 

## Vexriscv Processor

VexRiscv is an FPGA-friendly CPU core that implements the RISC-V instruction set architecture (ISA). Designed with flexibility and scalability in mind, it caters to a wide range of applications, from compact microcontroller-based systems to more complex multi-core configurations. The project is open-source under the MIT license, promoting community collaboration and adaptation. 

VexRiscv is developed using SpinalHDL, a high-level hardware description language that enhances design modularity and reusability. This approach allows for a plugin-based architecture, enabling users to customize and extend the CPU's capabilities to meet specific project requirements.

## RISC-V Minimal Viable Debugger

To create a minimum viable debugger, you need the following capabilities:

*   **Memory Access**: The ability to **peak and poke** memory. This means reading from and writing to specific memory locations.

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
    |   Memory Access      |----->| Memory                |
    |  (Read/Write)        |      |                       |
    +-----------------------+      +-----------------------+
             |                           |
             |                           |
    +-----------------------+      +-----------------------+
    | CPU Register Access  |----->| CPU Registers         |
    |  (Read/Write)        |      |                       |
    +-----------------------+      +-----------------------+
             |                           |
             |                           |
    +-----------------------+      +-----------------------+
    |   CPU Control         |----->| CPU (Halt, Step, etc.)|
    | (Halt, Step, Reset)   |      |                       |
    +-----------------------+      +-----------------------+
```

---

### VexRiscv Debugging

The VexRiscv soft core supports minimal debugging by utilizing a debug plugin, which has two registers that allow for complete control of the CPU.

*   The first register, Control Register, accessed at address zero, is a 32-bit register used to control the CPU. Through this register, it is possible to:
    *   Reset the CPU.
    *   Check if the CPU is halted.
    *   Single-step the CPU.
    *   Detect if the CPU has hit a breakpoint.

*   The second Instruction Injection Port is used to inject instructions into the CPU pipeline. By writing an instruction to this register, the CPU executes the instruction when it is paused.  The result of the instruction is then read from the same register. This allows for reading and writing to all of the CPU registers.
    *   For example, by injecting an instruction that moves a value from one register to another, the debugger can read the value of the source register by examining the destination register after execution.
    *   This same method can be used to construct memory addresses and write values to memory.

With these two registers, the core has all the necessary functions for a complete debugger, thus meeting the minimum requirements of a viable debugger.

---

### Comparison to RISC-V Debug Specification Methods

#### Memory Access Methods

To be compliant, the implementation must have at least one of the memory access mechanisms: Program Buffer, Abstract Access Memory or System Bus Access (SBA).

- **Program Buffer**: VexRiscv does not use a dedicated program buffer as specified in the RISC-V debug specification. Instead, it uses a single instruction injection port, where instructions are executed directly in the CPU pipeline. This offers similar capabilities to the RISC-V program buffer, but limited to a single instruction at a time.
- **Abstract Access Memory**: VexRiscv doesn’t directly implement an abstract access memory command. Instead, it uses its instruction injection port to achieve similar functionality, allowing the debugger to execute any memory access instruction and thus access memory from the hart's perspective.
- **System Bus Access**: VexRiscv does not implement a separate system bus access block. The instruction injection mechanism is used to access memory and registers, which provides similar functionality but with instructions executed by the CPU. This means it cannot use this method to access memory while the CPU is running, as described in the RISC-V debug specification.

#### Register Access

- VexRiscv's debug implementation accesses CSRs by injecting RISC-V instructions to read or write to these registers.
- RISC-V debug specification states the Control and Status Registers (CSRs) are accessed using the abstract access register, by or-ing the register number with 0x1000. CSRs can also be accessed using the program buffer, if abstract access is not support

#### Control Register

-  VexRiscv uses a single, custom control register for debug control, while the RISC-V debug specification employs a more standardized and distributed approach with multiple control registers.

---


### Summary

By addressing these points, VexRiscv can achieve full compliance with the RISC-V debug specification, ensuring greater interoperability with standard debugging tools and environments. The current custom implementation provides a way to debug VexRiscv, but it is not compliant with the specification.