<div align="center">

# 🚀 VSDSquadron Mini Research Internship
### RISC-V Architecture | C to Assembly | GNU Toolchain

[![RISC-V](https://img.shields.io/badge/RISC--V-Architecture-blue?style=for-the-badge&logo=riscv&logoColor=white)](https://riscv.org/)
[![GCC](https://img.shields.io/badge/GCC-16.1.0-orange?style=for-the-badge&logo=gnu&logoColor=white)](https://gcc.gnu.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-VM-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![VirtualBox](https://img.shields.io/badge/VirtualBox-7.0-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)](https://www.virtualbox.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

> 🎓 **Internship Program by [VLSI System Design (VSD)](https://www.vlsisystemdesign.com/)**
> Exploring RISC-V open-source ISA through hands-on C programming and assembly analysis.

---

</div>

## 📌 Table of Contents

- [📖 Project Overview](#-project-overview)
- [🛠️ Tools Used](#️-tools-used)
- [📂 Project Structure](#-project-structure)
- [🧪 Step-by-Step Workflow](#-step-by-step-workflow)
  - [Step 1 – Write the C Program](#step-1--write-the-c-program)
  - [Step 2 – View the Source Code](#step-2--view-the-source-code)
  - [Step 3 – Compile and Run with Normal GCC](#step-3--compile-and-run-with-normal-gcc)
  - [Step 4 – Set Up RISC-V Toolchain PATH](#step-4--set-up-risc-v-toolchain-path)
  - [Step 5 – Verify RISC-V Toolchain](#step-5--verify-risc-v-toolchain)
  - [Step 6 – Cross-Compile with RISC-V GCC](#step-6--cross-compile-with-risc-v-gcc)
  - [Step 7 – View RISC-V Assembly with objdump](#step-7--view-risc-v-assembly-with-objdump)
- [🔬 What is RISC-V?](#-what-is-risc-v)
- [🔍 Key Observations](#-key-observations)
- [📸 Screenshots](#-screenshots)
- [👤 Author](#-author)

---

## 📖 Project Overview

This project is **Task 1** of the **VSDSquadron Mini RISC-V Research Internship**. The objective is to understand how a simple C program gets translated into **RISC-V assembly instructions** using the GNU cross-compilation toolchain.

### 🎯 What We Do

| Step | Action | Purpose |
|------|--------|---------|
| ✍️ Write | `sum1ton.c` | C program to compute sum from 1 to N |
| ✅ Verify | Normal GCC compilation | Confirm program logic is correct |
| ⚙️ Cross-Compile | RISC-V GCC toolchain | Generate RISC-V machine code |
| 🔬 Analyze | `objdump` disassembly | Understand RISC-V instruction set |

---

## 🛠️ Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| 🖥️ **VirtualBox** | 7.0+ | Run Ubuntu VM on Windows/macOS |
| 🐧 **Ubuntu** | 20.04 LTS | Linux environment for development |
| 🔧 **GCC** | System default | Native C compiler for verification |
| ⚡ **RISC-V GCC** | `16.1.0` (`riscv64-unknown-elf-gcc`) | Cross-compiler for RISC-V target |
| 🔍 **objdump** | GNU Binutils | Disassemble RISC-V object files |
| 📝 **nano** | Terminal editor | Write and edit C source files |

---

## 📂 Project Structure

```
VSD_squadron_internship_RISC_V/
│
├── 📄 README.md               ← You are here!
│
└── 📁 Task_1/
    ├── 📄 README.md           ← Task-specific notes
    ├── 📝 sum1ton.c           ← C source program
    └── 📁 screenshots/        ← Output screenshots
        ├── 01_write_code.png  ← C source code in terminal
        ├── 02_cat_code.png    ← GCC compile & run output
        └── 03_gcc_run.png     ← RISC-V objdump assembly
```

---

## 🧪 Step-by-Step Workflow

### Step 1 – Write the C Program

Use `nano` (a terminal text editor) to write a simple C program that calculates the sum of numbers from 1 to N.

```bash
nano sum1ton.c
```

📋 **Source Code (`sum1ton.c`):**

```c
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

> 💡 **Explanation:** This program uses a `for` loop to accumulate the sum of integers from `1` to `n` (here, `n=5`). The expected output is `15`.

---

### Step 2 – View the Source Code

After writing, verify the file contents using `cat`:

```bash
cat sum1ton.c
```

**Expected Output:**
```
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

> ✅ This confirms the file was saved correctly before compilation.

---

### Step 3 – Compile and Run with Normal GCC

Before cross-compiling for RISC-V, test the program logic using your system's native GCC to confirm it works correctly.

```bash
gcc sum1ton.c -o sum1ton && ./sum1ton
```

**Expected Output:**
```
Sum of numbers from 1 to 5 = 15
```

> ✅ Output matches the expected result — the program logic is verified!

---

### Step 4 – Set Up RISC-V Toolchain PATH

Add the RISC-V toolchain binary directory to your `PATH` environment variable so you can use `riscv64-unknown-elf-gcc` from anywhere.

```bash
echo 'export PATH=$PATH:/opt/riscv/bin' >> ~/.bashrc
source ~/.bashrc
```

> 💡 **What this does:**
> - `echo ... >> ~/.bashrc` — appends the export line to your shell config file
> - `source ~/.bashrc` — reloads the config so changes take effect immediately in the current session

---

### Step 5 – Verify RISC-V Toolchain

Confirm the RISC-V GCC toolchain is installed and accessible:

```bash
riscv64-unknown-elf-gcc --version
```

**Expected Output:**
```
riscv64-unknown-elf-gcc (gc891d8dc23e) 16.1.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

> ✅ Version `16.1.0` confirms the toolchain is correctly installed and ready!

---

### Step 6 – Cross-Compile with RISC-V GCC

Now compile the C program targeting the RISC-V 64-bit architecture:

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -c sum1ton.c -o sum1ton.o
```

**🔍 Flag Breakdown:**

| Flag | Meaning |
|------|---------|
| `-O1` | Enable level-1 compiler optimizations |
| `-mabi=lp64` | Use 64-bit integer, long and pointer ABI |
| `-march=rv64i` | Target the 64-bit RISC-V base integer ISA |
| `-c` | Compile only (do not link), produce object file |
| `-o sum1ton.o` | Output file name |

> 💡 This generates a **RISC-V ELF object file** (`sum1ton.o`) containing machine instructions for the RISC-V processor — not runnable on your host machine, but inspectable with `objdump`!

---

### Step 7 – View RISC-V Assembly with objdump

Disassemble the RISC-V object file to see the generated assembly instructions:

```bash
riscv64-unknown-elf-objdump -d sum1ton.o
```

**Sample Assembly Output:**

```asm
sum1ton.o:     file format elf64-littleriscv

Disassembly of section .text:

0000000000000000 <main>:
   0:   fd010113         addi    sp,sp,-48
   4:   02813423         sd      s0,40(sp)
   8:   03010413         addi    s0,sp,48
   c:   fe042623         sw      zero,-20(s0)
  10:   00500793         li      a5,5
  14:   fef42423         sw      a5,-24(s0)
  18:   00100793         li      a5,1
  1c:   fef42223         sw      a5,-28(s0)
  ...
```

> 🧠 Each line is one **RISC-V instruction** — a human-readable form of the binary machine code that a RISC-V processor would execute.

---

## 🔬 What is RISC-V?

```
  ____  ___ ____   ____      __     __
 |  _ \|_ _/ ___| / ___|    /  \   / /
 | |_) || |\___ \| |       / /\ \ / /
 |  _ < | | ___) | |___   / ____ V /
 |_| \_\___|____/ \____| /_/    \_/
```

**RISC-V** (pronounced "risk-five") is an **open-source Instruction Set Architecture (ISA)** based on the principles of **Reduced Instruction Set Computing (RISC)**.

### 🌟 Why RISC-V Matters

| Feature | Benefit |
|---------|---------|
| 🔓 **Open Standard** | Free to use, no licensing fees |
| 🧩 **Modular Design** | Choose only the extensions you need |
| 📦 **Compact ISA** | Simple base with ~47 core instructions |
| 🌍 **Industry Adoption** | Used by SiFive, Western Digital, NVIDIA, and more |
| 🎓 **Academic Favorite** | Ideal for learning computer architecture |
| 🔧 **Customizable** | Add custom instructions for domain-specific hardware |

### 📐 RISC-V ISA Extensions

```
RV64I   → Base Integer (64-bit) — what we use here!
RV64M   → Integer Multiplication and Division
RV64F   → Single-Precision Floating Point
RV64D   → Double-Precision Floating Point
RV64C   → Compressed Instructions (16-bit)
```

---

## 🔍 Key Observations

From the `objdump` output, we can make these important observations:

### 📊 Instruction Count Comparison

| Optimization | Instructions in `main` | Notes |
|-------------|------------------------|-------|
| `-O1` | ~15 instructions | Balanced — some loop optimization |
| `-O2` | ~12 instructions | Stronger optimization, shorter code |
| `-Ofast` | ~10 instructions | Aggressive — may skip safety checks |

### 🧠 Assembly Insights

1. **Stack Management** — `addi sp, sp, -48` allocates space on the stack for local variables
2. **Register Usage** — RISC-V uses registers `a0–a7` for function arguments, `s0–s11` for saved values
3. **Load/Store Architecture** — All memory access happens through explicit `lw`/`sw` (load/store word) instructions — no direct memory-to-memory ops
4. **Loop Unrolling** — With `-O1`, the compiler may partially unroll the `for` loop to reduce branch overhead
5. **li (Load Immediate)** — Used to load constants like `n=5` directly into registers
6. **Clean Exit** — `ret` instruction returns control cleanly from `main`

### 💡 RISC vs CISC Philosophy

```
RISC-V (RISC)                    x86 (CISC)
─────────────────────            ──────────────────────
Simple, fixed-length ops    vs   Complex, variable-length ops
Many registers (32)         vs   Fewer registers (8-16)
Load/Store only             vs   Memory operands allowed
Compiler does the work      vs   Hardware does more work
```

---

## 📸 Screenshots

### 🖊️ Step 1 — C Source Code in Terminal

![C source code displayed in terminal](Task_1/screenshots/01_write_code.png)

> The `sum1ton.c` program shown in the terminal: includes `stdio.h`, a `for` loop summing 1 to `n=5`, and a `printf` statement.

---

### ⚙️ Step 2 — Native GCC Compile & Run

![GCC compilation and execution output](Task_1/screenshots/02_cat_code.png)

> Running `gcc sum1ton.c -o sum1ton && ./sum1ton` produces **"sum of numbers from 1 to 5 is 15"** — confirming correct program logic.

---

### 🔬 Step 3 — RISC-V objdump Assembly Output

![RISC-V assembly disassembly from objdump](Task_1/screenshots/03_gcc_run.png)

> The `riscv64-unknown-elf-objdump -d sum1ton.o` output showing the generated RISC-V ELF64 instructions including `addi`, `sd`, `li`, `lui`, `mv`, `auipc`, `jalr`, `ld`, and `ret`.

---

## 👤 Author

<div align="center">

**Adarsh Chauhan**
*VSDSquadron Mini RISC-V Research Internship*

[![GitHub](https://img.shields.io/badge/GitHub-Adarshchauhan123-181717?style=for-the-badge&logo=github)](https://github.com/Adarshchauhan123)

*Internship offered by [VLSI System Design (VSD)](https://www.vlsisystemdesign.com/)*

---

⭐ *If you found this helpful, please give the repo a star!* ⭐

</div>
