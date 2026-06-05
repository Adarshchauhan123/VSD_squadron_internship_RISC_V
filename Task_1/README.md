# 📁 Task 1 – RISC-V Toolchain & C Program Compilation

[![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)]()
[![Task](https://img.shields.io/badge/Task-1%20of%20N-blue?style=flat-square)]()

## 🎯 Objective

> Compile a simple C program (`sum1ton.c`) using the **RISC-V GCC cross-compiler** and analyze the generated RISC-V assembly instructions with `objdump`.

## 📝 1-Line Summary

> *Installed RISC-V GNU Toolchain (v16.1.0), cross-compiled a C sum program with `-O1 -mabi=lp64 -march=rv64i` flags, and analyzed the generated 64-bit RISC-V assembly using `objdump` to understand how C code maps to RISC-V instructions.*

---

## 📂 Files in This Folder

| File | Description |
|------|-------------|
| `sum1ton.c` | C source — calculates sum of integers 1 to N |
| `screenshots/` | Terminal output screenshots (to be uploaded) |

---

## ⚡ Quick Command Reference

```bash
# 1. Write the C program
nano sum1ton.c

# 2. Verify with native GCC
gcc sum1ton.c -o sum1ton && ./sum1ton

# 3. Set up RISC-V PATH
echo 'export PATH=$PATH:/opt/riscv/bin' >> ~/.bashrc
source ~/.bashrc

# 4. Verify RISC-V toolchain
riscv64-unknown-elf-gcc --version

# 5. Cross-compile for RISC-V
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -c sum1ton.c -o sum1ton.o

# 6. View RISC-V assembly
riscv64-unknown-elf-objdump -d sum1ton.o
```

---

📸 *Screenshots coming soon — will be added after video demonstration.*
