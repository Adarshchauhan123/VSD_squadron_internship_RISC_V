# SPI Master IP — User Guide

**IP Name:** SPI Master  
**Version:** 1.0  
**Target Device:** Lattice iCE40UP5K-SG48 (VSDSquadron FM)  
**Document Type:** IP User Guide  

---

## Table of Contents

1. [Overview](#1-overview)
2. [Features](#2-features)
3. [Block Diagram](#3-block-diagram)
4. [Signal Description](#4-signal-description)
5. [Programming Model](#5-programming-model)
6. [Clock and Timing](#6-clock-and-timing)
7. [Transfer State Machine](#7-transfer-state-machine)
8. [SPI Protocol Behavior](#8-spi-protocol-behavior-mode-0)
9. [Address Decode Architecture](#9-address-decode-architecture)
10. [Bus Collision Guard](#10-bus-collision-guard)
11. [Reset Behavior](#11-reset-behavior)
12. [Known Limitations](#12-known-limitations)

---

## 1. Overview

The **SPI Master** IP is a minimal, fully synchronous, memory-mapped peripheral that implements SPI Mode 0 (CPOL=0, CPHA=0) for use in the VSDSquadron FM RISC-V SoC. It is designed to integrate as a fourth peripheral alongside the existing GPIO and UART blocks, occupying four word-aligned registers in the IO address space.

The IP is a single Verilog file (`spi_master.v`) with no external sub-modules, intended for straightforward integration via `\`include` into `riscv.v`.

### Design Goals

- **Minimal footprint** — fits in 21% of iCE40UP5K LC resources with GPIO and UART included
- **Software-driven** — all control via register reads and writes; no DMA or interrupt hardware required
- **Correct bus sharing** — exports `spi_sel` to allow the SoC to suppress bus collisions on shared decode bits
- **Simulation-friendly** — full reset initialization prevents X-propagation in testbenches

---

## 2. Features

| Feature | Specification |
|---------|--------------|
| SPI Mode | Mode 0 only (CPOL=0, CPHA=0) |
| Transfer width | 8 bits per transaction |
| Bit order | MSB first |
| Clock source | System clock (12 MHz external crystal, pin 35) |
| Clock division | Programmable via CLKDIV field (8-bit) |
| SCLK range | `sys_clk / (2 × (CLKDIV+1))` |
| SCLK at 12 MHz | 6 MHz (CLKDIV=0) to ~23 kHz (CLKDIV=255) |
| Chip selects | 1 × active-low `CS_N` |
| Register interface | 4 × 32-bit word-aligned registers |
| Base address | `IO_BASE + 64` (`0x400040`) |
| Transfer initiation | Software write to `SPI_CTRL.START` (bit 1) |
| Transfer completion | Poll `SPI_STATUS.DONE` (bit 1) |
| Interrupt support | None — polling only |
| Reset type | Synchronous active-low (`resetn`) |
| Technology | Vendor-agnostic RTL (Verilog-2001) |
| Synthesis result | 1136 LC on iCE40UP5K (21%), max freq 19.54 MHz |

---

## 3. Block Diagram

```
                        ┌──────────────────────────────────────────────────┐
                        │                  spi_master.v                    │
                        │                                                  │
   clk  ───────────────►│                                                  │
   resetn ─────────────►│  ┌─────────────────────┐                        │
                        │  │   Address Decoder    │                        │
   isIO ───────────────►│  │                      │  sel_ctrl              │
   mem_wordaddr[29:0]──►│  │  wordaddr[7:0]==16?──┼──────────────►┐       │
   mem_wstrb ──────────►│  │  wordaddr[7:0]==17?──┼──────────────►│       │
   mem_rstrb ──────────►│  │  wordaddr[7:0]==18?──┼──────────────►│       │
   mem_wdata[31:0] ────►│  │  wordaddr[7:0]==19?──┼──────────────►│       │
                        │  └─────────────────────┘  spi_sel ◄────┘       │
                        │             │                   │               │
                        │             ▼                   └───────────────┼──► spi_sel (out)
                        │  ┌─────────────────────┐                        │
                        │  │   Register File      │                        │
                        │  │                      │                        │
                        │  │  SPI_CTRL  [15:0]   │                        │
                        │  │    EN (bit0)         │                        │
                        │  │    START (bit1)      │                        │
                        │  │    CLKDIV (bits15:8) │                        │
                        │  │                      │                        │
                        │  │  SPI_TXDATA [7:0]   │                        │
                        │  │  SPI_RXDATA [7:0]   │                        │
                        │  │  SPI_STATUS [1:0]   │                        │
                        │  │    BUSY (bit0)       │                        │
                        │  │    DONE (bit1)       │                        │
                        │  └─────────────────────┘                        │
                        │             │                                    │
                        │             ▼                                    │
                        │  ┌─────────────────────┐                        │
                        │  │  Clock Divider       │                        │
                        │  │  clk_cnt counter     │                        │
                        │  │  (counts 0→CLKDIV)  │                        │
                        │  └─────────────────────┘                        │
                        │             │                                    │
                        │             ▼                                    │
                        │  ┌─────────────────────┐                        │
                        │  │  8-bit Shift Engine  │                        │
                        │  │                      │                        │
                        │  │  shift_reg[7:0]      │◄─────── MISO ─────────┼◄─ MISO (in)
                        │  │  bit_cnt [3:0]       │                        │
                        │  │  MOSI = shift[7]     ├──────────────────────►│──► MOSI (out)
                        │  └─────────────────────┘                        │
                        │             │                                    │
                        │             ▼                                    │
                        │  ┌─────────────────────┐                        │
                        │  │  Control FSM         │◄──── busy/done ────   │
                        │  │  (implicit, inline)  ├──────────────────────►│──► SCLK (out)
                        │  │                      ├──────────────────────►│──► CS_N (out)
                        │  └─────────────────────┘                        │
                        │             │                                    │
                        │  ┌─────────────────────┐                        │
                        │  │  Read Data Mux       ├──────────────────────►│──► spi_rdata[31:0]
                        │  └─────────────────────┘                        │
                        └──────────────────────────────────────────────────┘

  External SPI Bus:
    SCLK ──────────────────────────────────────────────────────────────► SPI Slave
    MOSI ──────────────────────────────────────────────────────────────► SPI Slave
    MISO ◄──────────────────────────────────────────────────────────── SPI Slave
    CS_N ──────────────────────────────────────────────────────────────► SPI Slave
```

---

## 4. Signal Description

### Input Signals

| Signal | Width | Description |
|--------|-------|-------------|
| `clk` | 1 | System clock (12 MHz from external crystal, pin 35) |
| `resetn` | 1 | Active-low synchronous reset |
| `isIO` | 1 | Asserted by the CPU when the memory access targets the IO address space |
| `mem_wstrb` | 1 | Write strobe — asserted for one cycle during a CPU write |
| `mem_rstrb` | 1 | Read strobe — asserted for one cycle during a CPU read |
| `mem_wordaddr` | 30 | Word address of the memory access (byte address ÷ 4) |
| `mem_wdata` | 32 | Write data from CPU |
| `MISO` | 1 | SPI Master-In, Slave-Out — serial data from slave device |

### Output Signals

| Signal | Width | Description |
|--------|-------|-------------|
| `spi_rdata` | 32 | Read data returned to the CPU for SPI register reads |
| `spi_sel` | 1 | High when the current access targets any SPI register; used by `riscv.v` to suppress bus collisions |
| `SCLK` | 1 | SPI serial clock output to slave (idle low, Mode 0) |
| `MOSI` | 1 | SPI Master-Out, Slave-In — MSB of shift register, continuously driven |
| `CS_N` | 1 | Active-low chip select; asserted for the duration of a transfer |

---

## 5. Programming Model

### 5.1 Typical Transfer Sequence

```
1. Write CLKDIV and EN=1 to SPI_CTRL          (configure speed, enable IP)
2. Write TX byte to SPI_TXDATA                 (load data to shift register)
3. Write EN=1, START=1, CLKDIV to SPI_CTRL    (trigger transfer)
4. Poll SPI_STATUS until DONE (bit 1) = 1      (wait for completion)
5. Read SPI_RXDATA                             (retrieve received byte)
6. Write 0x2 to SPI_STATUS                    (clear DONE flag)
```

### 5.2 Software Pseudo-Code

```c
// Step 1+2: Configure and load TX data
IO_OUT(SPI_TXDATA, tx_byte);
// Step 3: Start transfer (CLKDIV=4, START=1, EN=1)
IO_OUT(SPI_CTRL, (4 << 8) | 0x03);
// Step 4: Poll
while (!(IO_IN(SPI_STATUS) & 0x2));
// Step 5: Read
uint8_t rx = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
// Step 6: Clear
IO_OUT(SPI_STATUS, 0x2);
```

### 5.3 START Bit Auto-Clear Behavior

The `START` bit (SPI_CTRL bit 1) is **write-only** in the sense that it auto-clears on the next rising clock edge after the transfer begins. Reading `SPI_CTRL` after a transfer is started will return `START=0`. This prevents accidental re-triggering on repeated reads.

**START fires only when:**
- `EN` (bit 0) = 1 in the same write
- `BUSY` = 0 (no transfer in progress)
- `START` (bit 1) = 1 in the write data

If any of these conditions is not met, the START write is ignored with no transfer initiated.

### 5.4 DONE Write-1-to-Clear Behavior

`SPI_STATUS.DONE` (bit 1) is set by hardware when a transfer completes and cleared by software by writing a `1` to that bit. Writing `0` to the DONE bit has no effect. This allows firmware to use read-modify-write patterns safely:

```c
IO_OUT(SPI_STATUS, 0x2);  // clears DONE only; BUSY is read-only
```

### 5.5 RXDATA Read-Only

`SPI_RXDATA` is read-only from the software perspective. Writes to offset 72 (word address 18) are silently ignored by the IP. The register updates internally when a transfer completes.

---

## 6. Clock and Timing

### 6.1 SCLK Frequency Formula

```
F_SCLK = F_SYS / (2 × (CLKDIV + 1))
```

Where `CLKDIV` is the 8-bit value in `SPI_CTRL[15:8]`.

### 6.2 SCLK Frequency Table (12 MHz system clock)

| CLKDIV | SCLK Frequency | Period | Bits/Second |
|--------|---------------|--------|-------------|
| 0      | 6.000 MHz     | 167 ns | 6,000,000   |
| 1      | 3.000 MHz     | 333 ns | 3,000,000   |
| 2      | 2.000 MHz     | 500 ns | 2,000,000   |
| 4      | 1.200 MHz     | 833 ns | 1,200,000   |
| 5      | 1.000 MHz     | 1.0 µs | 1,000,000   |
| 11     | 500 kHz       | 2.0 µs | 500,000     |
| 23     | 250 kHz       | 4.0 µs | 250,000     |
| 59     | 100 kHz       | 10 µs  | 100,000     |
| 119    | 50 kHz        | 20 µs  | 50,000      |
| 255    | ~23.4 kHz     | 42.7µs | 23,438      |

### 6.3 Transfer Duration

A complete 8-bit SPI transfer (including `CS_N` setup and hold provided by the bit counter) takes:

```
T_transfer = 8 × 2 × (CLKDIV + 1) × T_SYS
           = 16 × (CLKDIV + 1) clock cycles
```

At CLKDIV=4 and 12 MHz: `16 × 5 = 80 cycles = 6.67 µs`

---

## 7. Transfer State Machine

The IP uses an implicit (non-encoded) FSM implemented directly in the `busy` flag:

```
                  ┌────────────────────────────────────────────────────┐
                  │                                                    │
       resetn=0   ▼                                                    │
         ┌──────────────┐                                              │
         │    IDLE      │  CS_N=1, SCLK=0, MOSI=shift_reg[7]          │
         │  (busy=0)    │                                              │
         └──────────────┘                                              │
               │                                                       │
               │  sel_ctrl & mem_wstrb                                 │
               │  & START=1 & EN=1 & !busy                            │
               ▼                                                       │
         ┌──────────────┐                                              │
         │   CAPTURE    │  shift_reg ← tx_data                        │
         │   & START    │  bit_cnt ← 0, clk_cnt ← 0                   │
         │              │  CS_N ← 0, busy ← 1, SCLK ← 0              │
         └──────────────┘                                              │
               │                                                       │
               ▼                                                       │
         ┌──────────────────────────────────────────────────────────┐  │
         │   TRANSFER (busy=1, CS_N=0)                               │  │
         │                                                           │  │
         │   clk_cnt < CLKDIV:  clk_cnt++                           │  │
         │   clk_cnt >= CLKDIV: clk_cnt←0, toggle SCLK             │  │
         │                                                           │  │
         │   on rising SCLK (!SCLK before toggle):                  │  │
         │     if bit_cnt < 7:                                       │  │
         │       shift_reg ← {shift_reg[6:0], MISO}                 │  │
         │       bit_cnt++                                           │  │
         │     if bit_cnt == 7:                                      │  │
         │       rx_data ← {shift_reg[6:0], MISO}                   │  │
         │       busy←0, done←1, CS_N←1, SCLK←0                    │  │
         └──────────────────────────────────────────────────────────┘  │
               │                                                       │
               │  bit_cnt==7 and rising edge                           │
               └───────────────────────────────────────────────────────┘
                           returns to IDLE
```

**Key timing properties:**
- MOSI is always driven from `shift_reg[7]` — it is valid from the moment `shift_reg` is loaded and remains valid until the next load
- MISO is sampled on the rising edge of SCLK (Mode 0: CPHA=0, sample on first edge)
- CS_N is asserted (low) before the first SCLK edge and de-asserted (high) after the last MISO sample

---

## 8. SPI Protocol Behavior (Mode 0)

```
         CS_N  ─┐                                               ┌─
                └───────────────────────────────────────────────┘
                 │                                               │
         SCLK  ──┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌─
                  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
                  │   │   │   │   │   │   │   │   │   │   │   │
         MOSI  ──[B7]─[B7─[B6]─[B6─[B5]─[B5─[B4]─[B4─...─[B0─[B0]
                      ↑       ↑       ↑       ↑               ↑
         MISO  ──[S7]─[S7─[S6]─[S6─[S5]─[S5─[S4]─[S4─...─[S0─[S0]
                      ↑       ↑       ↑       ↑               ↑
                 Sample points (rising SCLK edges, CPHA=0)

  B7..B0 = TX data bits (MSB first)
  S7..S0 = RX data bits from slave (MSB first)
```

- **CPOL=0:** SCLK idle state is LOW
- **CPHA=0:** Data is sampled on the **first** (rising) clock edge; data changes on falling edges
- CS_N goes low before the first clock edge and high after the last sample

---

## 9. Address Decode Architecture

The IP uses a two-level decode scheme identical to the GPIO IP from Task-5:

```
Level 1: spi_sel = sel_ctrl | sel_txdata | sel_rxdata | sel_status
         → Asserted for any access to the SPI register window
         → Exported to riscv.v to prevent bus collisions

Level 2: Internal mux selects which register to read/write
         sel_ctrl   = isIO & (mem_wordaddr[7:0] == 8'd16)  ← byte offset 64
         sel_txdata = isIO & (mem_wordaddr[7:0] == 8'd17)  ← byte offset 68
         sel_rxdata = isIO & (mem_wordaddr[7:0] == 8'd18)  ← byte offset 72
         sel_status = isIO & (mem_wordaddr[7:0] == 8'd19)  ← byte offset 76
```

Word addresses 16–19 (decimal) were chosen because:
- Bit 4 is set for all four (`16 = 0x10`, `17 = 0x11`, `18 = 0x12`, `19 = 0x13`)
- They do not conflict with existing GPIO (word addrs 8–10) or UART (word addr 4) decode bits
- The 4-register window fits cleanly in a 2-bit sub-address space within the SPI module

---

## 10. Bus Collision Guard

The `spi_sel` output is the mechanism by which the SPI IP prevents write collisions on the shared IO bus. In `riscv.v`, the following decode bits overlap with SPI word addresses:

| Concern | Without guard | With guard |
|---------|--------------|------------|
| `uart_valid` | A write to `SPI_TXDATA` (word 17) would set `IO_UART_DAT_bit` and spuriously transmit a UART character | `& !spi_sel` suppresses `uart_valid` when SPI owns the bus |
| `LEDS` write | A write to SPI registers could toggle LEDs if address bits overlap | `& !spi_sel` suppresses the LED write path |

Both guards are added to `riscv.v` as part of integration (see `Integration_Guide.md`).

---

## 11. Reset Behavior

On `resetn = 0`, all internal registers are initialized to defined values:

| Register | Reset Value | Meaning |
|----------|-------------|---------|
| `en` | 0 | IP disabled |
| `clkdiv` | 4 | SCLK = 1.2 MHz at 12 MHz system clock |
| `tx_data` | 0x00 | No TX data |
| `rx_data` | 0x00 | No RX data |
| `busy` | 0 | Not busy |
| `done` | 0 | No completed transfer |
| `CS_N` | 1 | Chip deselected |
| `SCLK` | 0 | Clock idle low (Mode 0) |
| `bit_cnt` | 0 | Bit counter reset |
| `clk_cnt` | 0 | Clock divider counter reset |
| `shift_reg` | 0x00 | Shift register cleared |
| `sclk_en` | 0 | SCLK disabled |

> **Design Note:** Complete reset initialization is critical. Any register left uninitialized will start as `X` in simulation. Because `!busy` is a reset condition for START firing, an uninitialized `busy` (`X`) causes `!busy & X = X`, which Verilog treats as false — the state machine never starts.

---

## 12. Known Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| Single-byte transfers only | Cannot send multi-byte frames efficiently | Call the transfer function in a loop; re-assert CS_N manually is not possible (it's auto-controlled) |
| Mode 0 only (CPOL=0, CPHA=0) | Cannot communicate with SPI Mode 1/2/3 devices | Use only Mode 0 compatible sensors/peripherals |
| Polling only | CPU busy-waits during transfer | Acceptable for short 8-bit transfers at ≥100 kHz; use CLKDIV to keep transfer time < 100 µs |
| One CS_N | Cannot address multiple SPI slaves directly | Use GPIO pins as additional CS signals, controlled manually |
| No FIFO | Each byte must be individually started and polled | Write a helper function to handle the sequence |
| 12 MHz clock assumed | CLKDIV examples incorrect at other clock frequencies | Recalculate CLKDIV using `F_SCLK = F_SYS / (2 × (CLKDIV+1))` |
| No parity/CRC | No hardware error detection | Implement in firmware if required |
