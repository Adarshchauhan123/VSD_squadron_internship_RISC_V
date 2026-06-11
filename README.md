# VSDSquadron Mini Research Internship - RISC-V

Repository for the VSDSquadron Mini RISC-V Research Internship, encompassing hands-on tasks
covering C programming, RISC-V cross-compilation, simulation, and assembly-level debugging using
the GNU toolchain and Spike ISA simulator.

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
|   |-- README.md          <- Task 1 detailed documentation
|   |-- sum1ton.c          <- C source program
|   `-- screenshots/       <- Terminal output screenshots
|
`-- TASK_2/                <- Task 2: Spike simulation and register debugging
    |-- README.md          <- Task 2 detailed documentation
    `-- *.png              <- All task screenshots
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

## Environment Setup

| Component | Details |
|-----------|---------|
| Operating System | Ubuntu 20.04 LTS (running inside VirtualBox) |
| Host OS | Windows |
| GCC Version | System default |
| RISC-V Toolchain | riscv64-unknown-elf-gcc (gc891d8dc23e) 16.1.0 |
| Simulator | Spike RISC-V ISA Simulator |
| Text Editor | nano |

---

## About RISC-V

RISC-V is an open-source Instruction Set Architecture (ISA) based on the principles of Reduced
Instruction Set Computing. It is free to use with no licensing fees, modular in design, and
increasingly adopted across academic and industry applications for custom processor development.

The RISC-V 64-bit base integer ISA (RV64I) is used throughout this internship for all
cross-compilation and simulation tasks.
