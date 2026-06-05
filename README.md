# VSD Squadron Internship – RISC-V

## 🧑‍💻 Internship Overview

This repository documents the tasks completed as part of the **VSD Squadron Mini Internship** focused on **RISC-V Architecture** using the VSDSquadron Mini RISC-V development board.

---

## 📋 Task 1 – RISC-V Toolchain Installation & C Program Compilation

### 🎯 Objective

Install the RISC-V GNU Toolchain, compile a simple C program for the RISC-V target, and analyze the generated assembly instructions.

### 📝 Summary

> Installed the RISC-V GCC toolchain, wrote a basic C program (sum of numbers), compiled it using `riscv64-unknown-elf-gcc`, and examined the instruction count using `riscv64-unknown-elf-objdump` with both `-O1` and `-Ofast` optimization flags.

Screenshots and code will be uploaded after completion.

---

## 🛠️ Tools Used

| Tool | Description |
|------|-------------|
| `riscv64-unknown-elf-gcc` | RISC-V GCC cross-compiler |
| `riscv64-unknown-elf-objdump` | Disassembler for RISC-V ELF binaries |
| `spike` | RISC-V ISA Simulator |
| `pk` | Proxy kernel for RISC-V |
| Ubuntu 20.04 (via VirtualBox) | Host environment |

---

## 📂 Repository Structure

```
VSD_squadron_internship_RISC_V/
│
├── Task_1/
│   ├── README.md          ← This file
│   ├── sum1ton.c          ← Sample C program (sum of 1 to n)
│   └── screenshots/       ← Screenshots of output (to be uploaded)
│
└── README.md              ← Main repository overview
```

---

## 🔧 Steps Followed

### Step 1 – Install RISC-V Toolchain

```bash
sudo apt-get install git autoconf automake autotools-dev curl python3 libmpc-dev \
  libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool \
  patchutils bc zlib1g-dev libexpat-dev

git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
./configure --prefix=/opt/riscv
make
```

### Step 2 – Write a Simple C Program

```c
// sum1ton.c
#include <stdio.h>

int main() {
    int i, sum = 0, n = 5;
    for (i = 1; i <= n; i++) {
        sum += i;
    }
    printf("Sum of numbers from 1 to %d = %d\n", n, sum);
    return 0;
}
```

### Step 3 – Compile with RISC-V GCC

```bash
# Compile with -O1 optimization
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o sum1ton_O1.o sum1ton.c

# Compile with -Ofast optimization
riscv64-unknown-elf-gcc -Ofast -mabi=lp64 -march=rv64i -o sum1ton_Ofast.o sum1ton.c
```

### Step 4 – Disassemble and Count Instructions

```bash
# Disassemble with -O1
riscv64-unknown-elf-objdump -d sum1ton_O1.o | less

# Disassemble with -Ofast
riscv64-unknown-elf-objdump -d sum1ton_Ofast.o | less
```

### Step 5 – Observe Instruction Count Difference

| Optimization Flag | Instruction Count (main function) |
|-------------------|-----------------------------------|
| `-O1`             | ~15 instructions                  |
| `-Ofast`          | ~12 instructions                  |

> 📌 **Key Observation:** Using `-Ofast` reduces the number of instructions by aggressively optimizing loops and eliminating redundant operations.

---

## 📸 Screenshots

> *Screenshots will be uploaded after recording the demonstration video.*

---

## 👤 Author

**Adarsh Chauhan**  
VSD Squadron Mini RISC-V Internship  
GitHub: [Adarshchauhan123](https://github.com/Adarshchauhan123)

---

## 📄 License

This project is part of an academic internship program by [VLSI System Design (VSD)](https://www.vlsisystemdesign.com/).
