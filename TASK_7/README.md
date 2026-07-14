# SPI Master IP — VSDSquadron FPGA (iCE40UP5K-SG48)

> **Task-7 | VSD Squadron Internship — RISC-V IP Design**

---

## What Is This?

A fully synthesizable, memory-mapped **SPI Master** IP core for the **VSDSquadron FM FPGA** (Lattice iCE40UP5K-SG48). Plugs into the FemtoRV32-style RISC-V SoC (`riscv.v`) as a fourth peripheral alongside GPIO and UART.

| Property | Value |
|----------|-------|
| Mode | SPI Mode 0 (CPOL=0, CPHA=0) |
| Transfer Width | 8-bit, MSB-first |
| Bus | Memory-mapped, 32-bit word-aligned |
| System Clock | 12 MHz (external crystal, pin 35) |
| Base Address | `IO_BASE = 0x400000` |
| Registers | 4 (CTRL, TXDATA, RXDATA, STATUS) |
| Synthesis (iCE40UP5K) | 1136 LC (21%), 19.54 MHz max |
| Validated | Testbench ✅ + SoC Simulation ✅ + Hardware ✅ |

---

## 30-Second Quick Start

### 1. Copy the IP file
```bash
cp spi_master.v  vsdfpga_labs/basicRISCV/RTL/
```

### 2. Add io.h macros
```c
#define SPI_CTRL   64
#define SPI_TXDATA 68
#define SPI_RXDATA 72
#define SPI_STATUS 76
```

### 3. Send a byte and read the result
```c
IO_OUT(SPI_TXDATA, 0xA5);
IO_OUT(SPI_CTRL, (4 << 8) | 0x3);
while (!(IO_IN(SPI_STATUS) & 0x2));
IO_OUT(SPI_STATUS, 0x2);
uint8_t rx = IO_IN(SPI_RXDATA) & 0xFF;
```

### 4. Add SoC ports and rebuild
```bash
cd vsdfpga_labs/basicRISCV/RTL
make
sudo iceprog SOC.bin
```

---

## Register Map (Quick Reference)

| Register | Offset | Access | Key Fields |
|----------|--------|--------|------------|
| `SPI_CTRL`   | 64 (0x40) | R/W | Bit0=EN, Bit1=START, Bits[15:8]=CLKDIV |
| `SPI_TXDATA` | 68 (0x44) | R/W | Bits[7:0]=TX byte |
| `SPI_RXDATA` | 72 (0x48) | R   | Bits[7:0]=RX byte (read-only) |
| `SPI_STATUS` | 76 (0x4C) | R/W | Bit0=BUSY, Bit1=DONE (w1c) |

**SCLK frequency:** `sys_clk / (2 × (CLKDIV + 1))`  
At 12 MHz: CLKDIV=0 → 6 MHz, CLKDIV=4 → 1.2 MHz, CLKDIV=11 → 500 kHz

---

## Documentation

| Document | Description |
|----------|-------------|
| [docs/IP_User_Guide.md](docs/IP_User_Guide.md) | Full feature overview, block diagram, programming model, limitations |
| [docs/Register_Map.md](docs/Register_Map.md) | Complete register bit definitions, reset values, R/W behavior |
| [docs/Integration_Guide.md](docs/Integration_Guide.md) | Step-by-step SoC integration with all code changes |
| [docs/Example_Usage.md](docs/Example_Usage.md) | Ready-to-run C firmware examples |

---

## How to Test

### Standalone Testbench
```bash
cd vsdfpga_labs/basicRISCV/RTL
iverilog -o spi_tb spi_tb.v spi_master.v
vvp spi_tb
```
Expected: `RXDATA = 0xa5, PASS`

### Full-SoC Simulation
```bash
iverilog -DBENCH -o soc_sim riscv.v
timeout 120 vvp soc_sim > /tmp/spi_out.txt 2>&1
cat /tmp/spi_out.txt | tr -cd '[:print:]\n' | grep -v "^t=" | cut -c1 | \
    awk 'NR%2==1' | tr -d '\n' | sed 's/PASS/\nPASS\n/g'
```
Expected: `000000A5 PASS  0000003C PASS`

### Hardware Flash
```bash
cd vsdfpga_labs/basicRISCV/RTL
make
sudo iceprog SOC.bin
```
Expected: `VERIFY OK, cdone: high`

---

## Validation Summary

| Level | Test | Result |
|-------|------|--------|
| Standalone TB | 0xA5 MOSI→MISO loopback | ✅ PASS |
| Standalone TB | 0x3C MOSI→MISO loopback | ✅ PASS |
| SoC Simulation | 0xA5 + 0x3C UART output decoded | ✅ PASS |
| Hardware Synthesis | iCE40UP5K-SG48, 12 MHz | ✅ 21% LC, 19.54 MHz max |
| Hardware Flash | iceprog + readback verify | ✅ VERIFY OK |

---

## Files in This Package

```
TASK_7/
├── README.md
└── docs/
    ├── IP_User_Guide.md
    ├── Register_Map.md
    ├── Integration_Guide.md
    └── Example_Usage.md
```

---

## Known Limitations

- Single-byte only — no burst transfers, no DMA
- Mode 0 only — no software-selectable mode
- Polling only — no interrupt output
- One CS_N — no multi-slave support
- 12 MHz system clock assumed

---

*VSD Squadron Internship — RISC-V IP Design | Task-7 | Adarsh Chauhan*
