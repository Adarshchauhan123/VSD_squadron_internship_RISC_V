# SPI Master IP — User Guide

**IP Name:** SPI Master | **Version:** 1.0 | **Target:** Lattice iCE40UP5K-SG48 (VSDSquadron FM)

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

The **SPI Master** IP is a minimal, fully synchronous, memory-mapped peripheral implementing SPI Mode 0 (CPOL=0, CPHA=0) for use in the VSDSquadron FM RISC-V SoC. It integrates as a fourth peripheral alongside GPIO and UART, occupying four word-aligned registers in the IO address space.

Single Verilog file (`spi_master.v`), no external sub-modules, integrated via `` `include `` into `riscv.v`.

---

## 2. Features

| Feature | Specification |
|---------|--------------|
| SPI Mode | Mode 0 only (CPOL=0, CPHA=0) |
| Transfer width | 8 bits per transaction |
| Bit order | MSB first |
| Clock division | Programmable 8-bit CLKDIV field |
| SCLK range | `sys_clk / (2 × (CLKDIV+1))` |
| SCLK at 12 MHz | 6 MHz (CLKDIV=0) to ~23 kHz (CLKDIV=255) |
| Chip selects | 1 × active-low CS_N |
| Register interface | 4 × 32-bit word-aligned registers |
| Transfer initiation | Software write to SPI_CTRL.START (bit 1) |
| Transfer completion | Poll SPI_STATUS.DONE (bit 1) |
| Interrupt support | None — polling only |
| Reset type | Synchronous active-low (resetn) |
| Synthesis result | 1136 LC on iCE40UP5K (21%), max freq 19.54 MHz |

---

## 3. Block Diagram

```
                   ┌──────────────────────────────────────────────────┐
                   │                  spi_master.v                    │
                   │                                                  │
  clk  ───────────►│                                                  │
  resetn ─────────►│  ┌─────────────────────┐                        │
                   │  │   Address Decoder    │                        │
  isIO ───────────►│  │                      │                        │
  mem_wordaddr ───►│  │  wordaddr[7:0]==16?──┼──► sel_ctrl  ──►┐     │
  mem_wstrb ──────►│  │  wordaddr[7:0]==17?──┼──► sel_txdata ─►│     │
  mem_rstrb ──────►│  │  wordaddr[7:0]==18?──┼──► sel_rxdata ─►│     │
  mem_wdata ──────►│  │  wordaddr[7:0]==19?──┼──► sel_status ─►│     │
                   │  └─────────────────────┘    spi_sel ◄──────┘    │
                   │                                 │                │
                   │                                 └────────────────┼──► spi_sel (out)
                   │  ┌─────────────────────┐                         │
                   │  │   Register File      │                        │
                   │  │  SPI_CTRL  [15:8,0] │                        │
                   │  │  SPI_TXDATA [7:0]   │                        │
                   │  │  SPI_RXDATA [7:0]   │                        │
                   │  │  SPI_STATUS [1:0]   │                        │
                   │  └─────────────────────┘                        │
                   │             │                                    │
                   │  ┌─────────────────────┐                        │
                   │  │  Clock Divider       │                        │
                   │  │  clk_cnt → CLKDIV   │                        │
                   │  └─────────────────────┘                        │
                   │             │                                    │
                   │  ┌─────────────────────┐                        │
                   │  │  8-bit Shift Engine  │◄──────── MISO ────────┼◄─ MISO (in)
                   │  │  shift_reg[7:0]      │                        │
                   │  │  MOSI = shift[7]     ├────────────────────── ►│──► MOSI (out)
                   │  └─────────────────────┘                        │
                   │             │                                    │
                   │  ┌─────────────────────┐                        │
                   │  │  Control FSM         ├────────────────────── ►│──► SCLK (out)
                   │  │  busy / done flags   ├────────────────────── ►│──► CS_N (out)
                   │  └─────────────────────┘                        │
                   │  ┌─────────────────────┐                        │
                   │  │  Read Data Mux       ├────────────────────── ►│──► spi_rdata[31:0]
                   │  └─────────────────────┘                        │
                   └──────────────────────────────────────────────────┘
```

---

## 4. Signal Description

### Input Signals

| Signal | Width | Description |
|--------|-------|-------------|
| `clk` | 1 | System clock (12 MHz external crystal, pin 35) |
| `resetn` | 1 | Active-low synchronous reset |
| `isIO` | 1 | Asserted when the CPU targets IO address space |
| `mem_wstrb` | 1 | Write strobe — one cycle during CPU write |
| `mem_rstrb` | 1 | Read strobe — one cycle during CPU read |
| `mem_wordaddr` | 30 | Word address (byte address ÷ 4) |
| `mem_wdata` | 32 | Write data from CPU |
| `MISO` | 1 | SPI Master-In, Slave-Out |

### Output Signals

| Signal | Width | Description |
|--------|-------|-------------|
| `spi_rdata` | 32 | Read data returned to CPU |
| `spi_sel` | 1 | Asserted for any SPI register access; used by `riscv.v` to suppress bus collisions |
| `SCLK` | 1 | SPI serial clock (idle low, Mode 0) |
| `MOSI` | 1 | SPI Master-Out — MSB of shift register, continuously driven |
| `CS_N` | 1 | Active-low chip select; held low for entire transfer |

---

## 5. Programming Model

### Typical Transfer Sequence

```
1. Write TXDATA                    load TX byte
2. Write CTRL: CLKDIV, EN=1, START=1   trigger transfer
3. Poll STATUS until DONE=1        wait for completion
4. Read RXDATA                     retrieve received byte
5. Write 0x2 to STATUS             clear DONE flag
```

### Software Code

```c
IO_OUT(SPI_TXDATA, tx_byte);
IO_OUT(SPI_CTRL, (4 << 8) | 0x03);
while (!(IO_IN(SPI_STATUS) & 0x2));
IO_OUT(SPI_STATUS, 0x2);
uint8_t rx = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
```

### START Bit Behavior

START (SPI_CTRL bit 1) auto-clears after transfer begins. Fires only when EN=1, BUSY=0, and START=1 in the same write. Re-reading SPI_CTRL after a transfer returns START=0.

### DONE Write-1-to-Clear

Writing `0x2` to SPI_STATUS clears DONE. Writing `0x0` has no effect.

### RXDATA Read-Only

Writes to offset 72 are silently ignored. RXDATA updates when a transfer completes.

---

## 6. Clock and Timing

### SCLK Frequency Formula

```
F_SCLK = F_SYS / (2 × (CLKDIV + 1))
```

### SCLK Frequency Table (12 MHz system clock)

| CLKDIV | SCLK Frequency | Period |
|--------|---------------|--------|
| 0      | 6.000 MHz     | 167 ns |
| 1      | 3.000 MHz     | 333 ns |
| 2      | 2.000 MHz     | 500 ns |
| 4      | 1.200 MHz     | 833 ns |
| 5      | 1.000 MHz     | 1.0 µs |
| 11     | 500 kHz       | 2.0 µs |
| 23     | 250 kHz       | 4.0 µs |
| 59     | 100 kHz       | 10 µs  |
| 119    | 50 kHz        | 20 µs  |
| 255    | ~23.4 kHz     | 42.7µs |

### Transfer Duration

```
T_transfer = 16 × (CLKDIV + 1) system clock cycles

At CLKDIV=4, 12 MHz: 16 × 5 = 80 cycles = 6.67 µs
```

---

## 7. Transfer State Machine

```
  RESET  ──►  IDLE (busy=0, CS_N=1, SCLK=0)
                  │
                  │  START=1 & EN=1 & !busy
                  ▼
              TRANSFER (busy=1, CS_N=0)
                  │  clk_cnt counts 0→CLKDIV, toggles SCLK
                  │  On rising SCLK: shift MISO into shift_reg, bit_cnt++
                  │  On bit_cnt==7 (8th rising edge):
                  │    rx_data = {shift_reg[6:0], MISO}
                  │    busy=0, done=1, CS_N=1, SCLK=0
                  │
                  └──►  IDLE (done=1, RXDATA valid)
                              │
                              │  SW writes 0x2 to STATUS
                              ▼
                          IDLE (done=0, ready for next)
```

**Key:** MOSI = shift_reg[7] always. CS_N asserts before first SCLK, de-asserts after last sample.

---

## 8. SPI Protocol Behavior (Mode 0)

```
  CS_N  ─┐                                           ┌─
          └───────────────────────────────────────────┘
  SCLK  ───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
            └───┘   └───┘   └───┘   └───┘   └───┘   └──
            │   ↑   │   ↑   │   ↑   │   ↑           ↑
  MOSI  ──[B7]──[B6]──[B5]──[B4]──────────────────[B0]
  MISO  ──[S7]──[S6]──[S5]──[S4]──────────────────[S0]
              ↑       ↑       ↑       ↑           ↑
              Sample points (rising SCLK edges, CPHA=0)
```

CPOL=0: SCLK idle low. CPHA=0: sample on first (rising) edge.

---

## 9. Address Decode Architecture

```
Level 1: spi_sel = sel_ctrl | sel_txdata | sel_rxdata | sel_status
         Exported to riscv.v to prevent bus collisions.

Level 2: sel_ctrl   = isIO & (mem_wordaddr[7:0] == 8'd16)  offset 64
         sel_txdata = isIO & (mem_wordaddr[7:0] == 8'd17)  offset 68
         sel_rxdata = isIO & (mem_wordaddr[7:0] == 8'd18)  offset 72
         sel_status = isIO & (mem_wordaddr[7:0] == 8'd19)  offset 76
```

Word addresses 16–19 do not conflict with GPIO (8–10) or UART (4) decode bits.

---

## 10. Bus Collision Guard

`spi_sel` prevents write collisions on the shared IO bus:

| Concern | Without guard | With `& !spi_sel` |
|---------|--------------|-------------------|
| `uart_valid` | Write to SPI_TXDATA (word 17) spuriously transmits UART character | Suppressed |
| `LEDS` write | Write to SPI registers toggles LEDs | Suppressed |

---

## 11. Reset Behavior

All registers initialized synchronously on `resetn = 0`:

| Register | Reset Value |
|----------|-------------|
| `en` | 0 |
| `clkdiv` | 4 (default 1.2 MHz SCLK) |
| `tx_data`, `rx_data`, `shift_reg` | 0x00 |
| `busy`, `done`, `sclk_en` | 0 |
| `CS_N` | 1 (deselected) |
| `SCLK` | 0 (idle low) |
| `bit_cnt`, `clk_cnt` | 0 |

Complete reset initialization is required. An uninitialized `busy` (value X) causes `!busy & X = X`, which Verilog treats as false — START never fires.

---

## 12. Known Limitations

| Limitation | Workaround |
|-----------|------------|
| Single-byte transfers only | Loop the transfer function for multi-byte |
| Mode 0 only | Use only Mode 0 compatible peripherals |
| Polling only | Acceptable for 8-bit transfers; keep CLKDIV high enough for < 100 µs transfer |
| One CS_N | Use GPIO pins as additional CS signals |
| No FIFO | Write a helper function to handle the sequence |
| 12 MHz clock assumed | Recalculate CLKDIV: `CLKDIV = F_SYS/(2*F_SCLK) - 1` |
