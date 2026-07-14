# VSDSquadron Mini Research Internship - RISC-V

Repository for the VSDSquadron Mini RISC-V Research Internship, encompassing hands-on tasks
covering C programming, RISC-V cross-compilation, simulation, assembly-level debugging, GPIO/UART
peripheral design, and SPI Master IP integration using the GNU toolchain, Spike ISA simulator,
and Lattice iCE40UP5K-SG48 FPGA.

**Submitted by:** Adarsh Chauhan

**Roll No / Email ID:** 23uec504

**GitHub:** [Adarshchauhan123](https://github.com/Adarshchauhan123)

**Internship offered by:** [VLSI System Design (VSD)](https://www.vlsisystemdesign.com/)

---

## Repository Structure

```
VSD_squadron_internship_RISC_V/
|
|-- README.md              <- You are here (Introduction page)
|
|-- TASK_1/                <- Task 1: C compilation and RISC-V assembly analysis
|   |-- README.md
|   |-- sum1ton.c
|   `-- screenshots/
|
|-- TASK_2/                <- Task 2: Spike simulation and register debugging
|   |-- README.md
|   `-- *.png
|
|-- TASK_3/                <- Task 3: Custom RISC-V instructions
|   `-- README.md
|
|-- TASK_4/                <- Task 4: RISC-V functional simulation
|   `-- README.md
|
|-- TASK_5/                <- Task 5: Multi-register GPIO IP
|   `-- README.md
|
|-- TASK_6/                <- Task 6: SPI Master IP design and SoC integration
|   |-- README.md
|   `-- screenshots/
|
`-- TASK_7/                <- Task 7: SPI Master IP commercial documentation package
    |-- README.md
    `-- docs/
        |-- IP_User_Guide.md
        |-- Register_Map.md
        |-- Integration_Guide.md
        `-- Example_Usage.md
```

---

## Task 1

### Objective

To compile a simple C program that computes the sum of numbers from 1 to N using both the native
GCC compiler and the RISC-V GCC cross-compiler. To understand the compilation flow and analyze
the generated RISC-V assembly instructions using objdump.

### Program

Sum of integers from 1 to N (`sum1ton.c`), where N = 5.

Expected output: `sum of numbers from 1 to 5 is 15`

### Tools Used

| Tool | Purpose |
|------|---------|
| GCC | Native C compiler for x86-64 verification |
| riscv64-unknown-elf-gcc | RISC-V cross-compiler |
| riscv64-unknown-elf-objdump | RISC-V assembly disassembler |

### Key Steps

1. Write the C program using `nano`
2. Verify with native GCC compilation and execution
3. Cross-compile for RISC-V using the RISC-V GCC toolchain
4. Inspect generated RISC-V assembly using `objdump`

[View Task 1 Documentation](TASK_1/README.md)

---

## Task 2

### Objective

To write two custom C programs, cross-compile them for the RISC-V 64-bit architecture, simulate
execution using the Spike RISC-V ISA simulator, inspect generated assembly using objdump, and
debug register values step by step using the Spike interactive debugger.

### Programs

- Program 1: Sum of numbers from 1 to 100 (`sum1ton.c`)
  - Expected output: `sum of numbers from 1 to 100 is 5050`
- Program 2: Factorial of 10 (`factorial.c`)
  - Expected output: `Factorial of 10 is 3628800`

### Tools Used

| Tool | Purpose |
|------|---------|
| GCC | Native C compiler for x86-64 verification |
| riscv64-unknown-elf-gcc | RISC-V cross-compiler with -Ofast optimization |
| Spike | RISC-V ISA simulator |
| pk (proxy kernel) | Handles system calls during Spike simulation |
| riscv64-unknown-elf-objdump | RISC-V assembly disassembler |

### Key Steps

1. Write and verify both C programs using GCC
2. Cross-compile both programs for RISC-V 64-bit using -Ofast
3. Simulate execution on Spike with proxy kernel
4. Disassemble RISC-V binaries using objdump
5. Debug register state step by step using Spike interactive debugger

[View Task 2 Documentation](TASK_2/README.md)

---

## Task 3

### Objective

To understand and implement custom RISC-V instructions by extending the base ISA, and to simulate
the resulting behaviour using functional simulation tools.

### Key Steps

1. Identify the custom instruction encoding within the RISC-V ISA space
2. Implement the custom instruction in RTL
3. Verify functional behaviour through simulation
4. Analyze waveforms and register outputs

[View Task 3 Documentation](TASK_3/README.md)

---

## Task 4

### Objective

To perform functional simulation of a RISC-V processor core, verify instruction execution
correctness, and analyze the output waveforms to confirm expected register and memory behaviour.

### Tools Used

| Tool | Purpose |
|------|---------|
| iverilog | Verilog simulation |
| GTKWave | Waveform viewer |
| riscv64-unknown-elf-gcc | RISC-V firmware compiler |

### Key Steps

1. Write or obtain a RISC-V RTL core
2. Compile test firmware and generate hex
3. Run simulation with iverilog
4. Inspect waveforms in GTKWave

[View Task 4 Documentation](TASK_4/README.md)

---

## Task 5

### Objective

To design and integrate a multi-register GPIO IP into the FemtoRV32-style RISC-V SoC running on
the VSDSquadron FM FPGA (iCE40UP5K-SG48), exposing output data, direction, and read-back
registers through the memory-mapped IO bus.

### Registers Added

| Register | Offset | Description |
|----------|--------|-------------|
| `IO_GPIO_DATA` | 32 | Output data register |
| `IO_GPIO_DIR` | 36 | Direction register (1=output) |
| `IO_GPIO_READ` | 40 | Input read-back register |

### Key Steps

1. Design `gpio_ip.v` with three memory-mapped registers
2. Integrate into `riscv.v` alongside existing UART
3. Export `gpio_sel` to guard UART decode collisions
4. Update PCF and synthesize for iCE40UP5K-SG48
5. Verify with firmware and hardware flash

[View Task 5 Documentation](TASK_5/README.md)

---

## Task 6

### Objective

To design a minimal SPI Master IP (Mode 0, single-byte, MSB-first) and integrate it into the
existing FemtoRV32-style RISC-V SoC as a memory-mapped peripheral on the VSDSquadron FM FPGA.
Validate at three levels: standalone testbench, full-SoC simulation, and hardware synthesis.

### What Was Added

| Item | Detail |
|------|--------|
| New IP file | `spi_master.v` — synchronous SPI Master, Mode 0 |
| Registers | SPI_CTRL (offset 64), SPI_TXDATA (68), SPI_RXDATA (72), SPI_STATUS (76) |
| SoC changes | 6 edits to `riscv.v` — include, ports, instantiation, bus guards, mux |
| New pins | SCLK (pin 6), MOSI (pin 9), MISO (pin 10), CS_N (pin 11) |
| Synthesis | 21% LC (1136/5280), 19.54 MHz max on iCE40UP5K-SG48 |
| Result | VERIFY OK, cdone: high on hardware flash |

### Validation Results

| Level | Result |
|-------|--------|
| Standalone testbench | RXDATA = 0xA5, PASS |
| Full-SoC simulation | 0xA5 and 0x3C both PASS |
| Hardware flash | VERIFY OK, cdone: high |

[View Task 6 Documentation](TASK_6/README.md)

---

## Task 7

### Objective

To produce a complete commercial-grade IP documentation package for the SPI Master IP designed
in Task 6, covering user guide, register map, integration guide, and ready-to-run firmware
examples — structured as a professional IP datasheet deliverable.

### Documents Produced

| Document | Description |
|----------|-------------|
| `README.md` | Quick-start: what it is, how to integrate, how to test |
| `docs/IP_User_Guide.md` | Block diagram, programming model, FSM, clock table, limitations |
| `docs/Register_Map.md` | Complete register bit definitions, reset values, R/W behavior |
| `docs/Integration_Guide.md` | Step-by-step SoC integration with all code changes and commands |
| `docs/Example_Usage.md` | Ready-to-run C code examples including loopback, ADC, LED driver |

### Key Highlights

- ASCII block diagram of full datapath
- SCLK frequency table from 6 MHz down to 23 kHz (all CLKDIV values)
- Complete `spi_master.v` RTL with all 6 `riscv.v` integration edits shown
- 7 C firmware examples including W25Q16 flash, MCP3002 ADC, MAX7219 LED driver
- Full standalone testbench (`spi_tb.v`) with 6-pattern test loop

[View Task 7 Documentation](TASK_7/README.md)

---

## Environment Setup

| Component | Details |
|-----------|---------|
| Operating System | Ubuntu 20.04 LTS / GitHub Codespaces |
| Host OS | Windows |
| GCC Version | System default |
| RISC-V Toolchain | riscv64-unknown-elf-gcc |
| Simulator | Spike RISC-V ISA Simulator / iverilog |
| FPGA | Lattice iCE40UP5K-SG48 (VSDSquadron FM) |
| Synthesis | Yosys + nextpnr-ice40 + icestorm |
| Text Editor | nano / VS Code |

---

## About RISC-V

RISC-V is an open-source Instruction Set Architecture (ISA) based on the principles of Reduced
Instruction Set Computing. It is free to use with no licensing fees, modular in design, and
increasingly adopted across academic and industry applications for custom processor development.

The RISC-V 32-bit base integer ISA (RV32I) is used throughout the FPGA tasks (Tasks 4–7) via
the FemtoRV32 soft-core, while the 64-bit ISA (RV64I) is used in simulation tasks (Tasks 1–3).
