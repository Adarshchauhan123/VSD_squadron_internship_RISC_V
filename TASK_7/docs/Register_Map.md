# SPI Master IP — Register Map

**IP Name:** SPI Master  
**Version:** 1.0  
**Base Address:** `IO_BASE = 0x400000`  
**Register Width:** 32 bits (word-aligned)  
**Access Width:** 32-bit word only  

---

## Table of Contents

1. [Register Summary](#1-register-summary)
2. [Address Map](#2-address-map)
3. [SPI_CTRL — Control Register](#3-spi_ctrl--control-register-offset-64-0x40)
4. [SPI_TXDATA — Transmit Data Register](#4-spi_txdata--transmit-data-register-offset-68-0x44)
5. [SPI_RXDATA — Receive Data Register](#5-spi_rxdata--receive-data-register-offset-72-0x48)
6. [SPI_STATUS — Status Register](#6-spi_status--status-register-offset-76-0x4c)
7. [Register Access Rules](#7-register-access-rules)
8. [Reset Values Summary](#8-reset-values-summary)
9. [Bit Field Encoding Reference](#9-bit-field-encoding-reference)

---

## 1. Register Summary

| Register | Symbol | Byte Offset | Word Address | Absolute Address | Access | Reset |
|----------|--------|-------------|--------------|-----------------|--------|-------|
| Control Register | `SPI_CTRL` | 64 (0x40) | 16 (0x10) | 0x400040 | R/W | 0x00000000 |
| Transmit Data | `SPI_TXDATA` | 68 (0x44) | 17 (0x11) | 0x400044 | R/W | 0x00000000 |
| Receive Data | `SPI_RXDATA` | 72 (0x48) | 18 (0x12) | 0x400048 | R | 0x00000000 |
| Status Register | `SPI_STATUS` | 76 (0x4C) | 19 (0x13) | 0x40004C | R/W | 0x00000000 |

**Address computation:**
```
Absolute address = IO_BASE + Byte Offset
               = 0x400000 + offset

Word address = Byte Offset ÷ 4
```

**io.h macros:**
```c
#define IO_BASE    0x400000
#define SPI_CTRL   64     // word addr 16, absolute 0x400040
#define SPI_TXDATA 68     // word addr 17, absolute 0x400044
#define SPI_RXDATA 72     // word addr 18, absolute 0x400048
#define SPI_STATUS 76     // word addr 19, absolute 0x40004C

#define IO_IN(port)       *(volatile uint32_t*)(IO_BASE + port)
#define IO_OUT(port,val)  *(volatile uint32_t*)(IO_BASE + port)=(val)
```

---

## 2. Address Map

```
IO_BASE = 0x400000
│
├── 0x400000 + 0x04 = 0x400004  →  IO_LEDS       (existing)
├── 0x400000 + 0x08 = 0x400008  →  IO_UART_DAT   (existing)
├── 0x400000 + 0x10 = 0x400010  →  IO_UART_CNTL  (existing)
├── 0x400000 + 0x20 = 0x400020  →  IO_GPIO_DATA  (existing, Task-5)
├── 0x400000 + 0x24 = 0x400024  →  IO_GPIO_DIR   (existing, Task-5)
├── 0x400000 + 0x28 = 0x400028  →  IO_GPIO_READ  (existing, Task-5)
│
├── 0x400000 + 0x40 = 0x400040  →  SPI_CTRL      ← NEW (Task-7)
├── 0x400000 + 0x44 = 0x400044  →  SPI_TXDATA    ← NEW (Task-7)
├── 0x400000 + 0x48 = 0x400048  →  SPI_RXDATA    ← NEW (Task-7)
└── 0x400000 + 0x4C = 0x40004C  →  SPI_STATUS    ← NEW (Task-7)
```

---

## 3. SPI_CTRL — Control Register (Offset 64, 0x40)

**Absolute address:** `0x400040`  
**Word address:** `16` (`0x10`)  
**Access:** Read/Write  
**Reset value:** `0x00000000`

### Bit Field Diagram

```
 Bit:   31      16  15       8   7    2   1    0
        ┌──────────┬──────────┬────────┬────┬────┐
        │ Reserved │  CLKDIV  │  Rsvd  │ ST │ EN │
        │  [31:16] │  [15:8]  │  [7:2] │[1] │[0] │
        └──────────┴──────────┴────────┴────┴────┘
          (read 0)   R/W          R/W    W   R/W
```

### Bit Definitions

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:16] | Reserved | R | 0 | Read as zero. Do not write 1 to these bits. |
| [15:8] | `CLKDIV` | R/W | 0x00 | SPI clock divider. `F_SCLK = F_SYS / (2 × (CLKDIV+1))`. At 12 MHz: CLKDIV=0→6MHz, CLKDIV=4→1.2MHz, CLKDIV=11→500kHz. |
| [7:2] | Reserved | R | 0 | Read as zero. |
| [1] | `START` | W (auto-clear) | 0 | Write 1 to initiate a transfer. Transfer starts only if `EN=1` and `BUSY=0` in the same write. Auto-clears to 0 after transfer begins. Reading this bit always returns 0. |
| [0] | `EN` | R/W | 0 | Enable the SPI IP. Must be 1 for any transfer to proceed. Write 0 to disable the IP. |

### Register Write Behavior

Writing to `SPI_CTRL` performs these actions simultaneously:
1. Updates `EN` from write data bit 0
2. Updates `CLKDIV` from write data bits [15:8]
3. If `START=1` AND `EN=1` (in the write data) AND `BUSY=0` (current state):
   - Loads `shift_reg` from `tx_data`
   - Resets `bit_cnt`, `clk_cnt`
   - Asserts `CS_N = 0`
   - Sets `busy = 1`
   - Sets `SCLK = 0`
   - Begins clock division counting

### Programming Examples

```c
// Configure CLKDIV=4, enable, start transfer in one write
IO_OUT(SPI_CTRL, (4 << 8) | 0x03);   // CLKDIV=4, START=1, EN=1

// Change clock speed only (no start)
IO_OUT(SPI_CTRL, (11 << 8) | 0x01);  // CLKDIV=11 (500kHz), EN=1, START=0

// Disable the IP
IO_OUT(SPI_CTRL, 0x00);               // EN=0
```

---

## 4. SPI_TXDATA — Transmit Data Register (Offset 68, 0x44)

**Absolute address:** `0x400044`  
**Word address:** `17` (`0x11`)  
**Access:** Read/Write (writes load TX data; reads return last written value)  
**Reset value:** `0x00000000`

### Bit Field Diagram

```
 Bit:   31              8   7            0
        ┌───────────────────┬────────────┐
        │     Reserved      │   TXDATA   │
        │      [31:8]       │   [7:0]    │
        └───────────────────┴────────────┘
           (read 0, ign. wr)    R/W
```

### Bit Definitions

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:8] | Reserved | R | 0 | Read as zero. Writes to these bits are ignored. |
| [7:0] | `TXDATA` | R/W | 0x00 | Byte to transmit in the next SPI transfer. Write before asserting `START` in `SPI_CTRL`. MSB (bit 7) is transmitted first. Value is held until next write or reset. |

### Write Timing

```
         clk  ─┬─┬─┬─┬─┬─┬─┬─┬─
                │ │ │ │ │ │ │ │
  sel_txdata  ──┐ │ └─                (asserted one cycle)
  mem_wstrb   ──┘                    (write strobe)
                │
  tx_data reg ──┴──────────────────── (latched on posedge clk when sel_txdata & mem_wstrb)
```

`tx_data` is latched **synchronously** on the rising clock edge when `sel_txdata & mem_wstrb` is true. Writing `SPI_TXDATA` and `SPI_CTRL.START` in the same bus transaction is not supported (they are separate memory-mapped addresses). Always write `TXDATA` before writing `CTRL` with `START=1`.

### Programming Example

```c
// Load byte 0xA5 for transmission
IO_OUT(SPI_TXDATA, 0xA5);
// Now trigger the transfer
IO_OUT(SPI_CTRL, (4 << 8) | 0x03);
```

---

## 5. SPI_RXDATA — Receive Data Register (Offset 72, 0x48)

**Absolute address:** `0x400048`  
**Word address:** `18` (`0x12`)  
**Access:** Read-only (writes are silently ignored)  
**Reset value:** `0x00000000`  
**Updated:** At end of each completed transfer

### Bit Field Diagram

```
 Bit:   31              8   7            0
        ┌───────────────────┬────────────┐
        │     Reserved      │   RXDATA   │
        │      [31:8]       │   [7:0]    │
        └───────────────────┴────────────┘
           (always 0)          R (only)
```

### Bit Definitions

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:8] | Reserved | R | 0 | Always reads as zero. |
| [7:0] | `RXDATA` | R | 0x00 | Byte received from the SPI slave during the last completed transfer. Bit 7 is the first bit received (MSB first). Valid after `SPI_STATUS.DONE=1`. Holds value until next transfer completes. |

### When is RXDATA Valid?

```
Transfer starts  →  Transfer completes  →  DONE=1 set  →  Read RXDATA
                                                            ↑ valid here
```

Reading `SPI_RXDATA` while `BUSY=1` returns the value from the **previous** transfer (or 0x00 after reset). Always check `DONE=1` before reading.

### Programming Example

```c
// Wait for transfer completion
while (!(IO_IN(SPI_STATUS) & 0x2));
// Now read valid RX data
uint8_t received = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
```

---

## 6. SPI_STATUS — Status Register (Offset 76, 0x4C)

**Absolute address:** `0x40004C`  
**Word address:** `19` (`0x13`)  
**Access:** Read/Write (DONE bit is write-1-to-clear; BUSY is read-only)  
**Reset value:** `0x00000000`

### Bit Field Diagram

```
 Bit:   31               2   1     0
        ┌─────────────────────┬──────┬──────┐
        │       Reserved      │ DONE │ BUSY │
        │        [31:2]       │ [1]  │ [0]  │
        └─────────────────────┴──────┴──────┘
              (read 0)          R/W1C   R
```

### Bit Definitions

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:2] | Reserved | R | 0 | Read as zero. Do not write 1 to these bits. |
| [1] | `DONE` | R/W1C | 0 | **Transfer Complete flag.** Set by hardware when the 8-bit transfer finishes (`bit_cnt==7` on the 8th rising SCLK edge). Clear by writing 1 to this bit. Writing 0 has no effect. `RXDATA` is valid when `DONE=1`. |
| [0] | `BUSY` | R | 0 | **Transfer Active flag.** Set by hardware when `START` fires. Cleared when transfer completes. Software cannot write this bit (writes are ignored). A new transfer cannot start while `BUSY=1`. |

### Status Bit State Machine

```
                  ┌─────────────────────────────────────┐
                  │                                     │
     RESET ──────►│  BUSY=0, DONE=0                     │
                  │  (idle, no transfer in progress)    │
                  └─────────────────────────────────────┘
                               │
                    START fires (HW)
                               │
                               ▼
                  ┌─────────────────────────────────────┐
                  │  BUSY=1, DONE=0                     │
                  │  (transfer in progress)             │
                  └─────────────────────────────────────┘
                               │
                    Transfer completes (HW)
                               │
                               ▼
                  ┌─────────────────────────────────────┐
                  │  BUSY=0, DONE=1                     │
                  │  (transfer complete, RXDATA valid)  │
                  └─────────────────────────────────────┘
                               │
                    SW writes 0x2 to SPI_STATUS
                               │
                               ▼
                  ┌─────────────────────────────────────┐
                  │  BUSY=0, DONE=0                     │
                  │  (idle, ready for next transfer)    │
                  └─────────────────────────────────────┘
```

### Programming Examples

```c
// Poll for completion
while (!(IO_IN(SPI_STATUS) & 0x2));   // wait for DONE=1

// Check if transfer is in progress
if (IO_IN(SPI_STATUS) & 0x1) {        // BUSY=1
    // cannot start new transfer
}

// Clear DONE flag (write-1-to-clear)
IO_OUT(SPI_STATUS, 0x2);              // clears DONE, BUSY unchanged (read-only)

// Full status check
uint32_t status = IO_IN(SPI_STATUS);
if (status & 0x1) { /* busy */ }
if (status & 0x2) { /* done */ }
```

---

## 7. Register Access Rules

| Register | Write Effect | Read Returns |
|----------|-------------|-------------|
| `SPI_CTRL` | Updates `EN`, `CLKDIV`; if `START=1 & EN=1 & !BUSY`, initiates transfer | Current `CLKDIV` in [15:8], current `EN` in [0]; bit 1 always reads 0 |
| `SPI_TXDATA` | Stores byte [7:0] in `tx_data` holding register | Last written value in [7:0], zeros in [31:8] |
| `SPI_RXDATA` | **Write ignored** — read-only register | Last received byte in [7:0] after transfer, 0x00 after reset |
| `SPI_STATUS` | Bit 1 (DONE) only: write 1 clears DONE; all other bits: write ignored | Current `DONE` in [1], current `BUSY` in [0] |

### Critical Ordering Rule

```
ALWAYS write TXDATA before writing CTRL with START=1.
The SPI IP latches tx_data into shift_reg at the moment START fires.
If TXDATA is written after START, the transfer will use the PREVIOUS tx_data value.
```

### Simultaneous Read/Write

Simultaneous reads and writes to the same register are not supported by the bus interface. The `riscv.v` memory bus does not assert `mem_wstrb` and `mem_rstrb` simultaneously.

---

## 8. Reset Values Summary

| Register | Field | Reset Value | Notes |
|----------|-------|-------------|-------|
| `SPI_CTRL` | CLKDIV [15:8] | 0x04 | Default SCLK = 12MHz/(2×5) = 1.2 MHz |
| `SPI_CTRL` | START [1] | 0 | No pending transfer |
| `SPI_CTRL` | EN [0] | 0 | IP disabled at reset |
| `SPI_TXDATA` | TXDATA [7:0] | 0x00 | No TX data |
| `SPI_RXDATA` | RXDATA [7:0] | 0x00 | No RX data yet |
| `SPI_STATUS` | DONE [1] | 0 | No completed transfer |
| `SPI_STATUS` | BUSY [0] | 0 | Not busy |

> **Note:** `CLKDIV` resets to 4, not 0. This gives a safe 1.2 MHz default SCLK at 12 MHz system clock, suitable for most SPI peripherals.

---

## 9. Bit Field Encoding Reference

### SPI_CTRL Full Encoding Examples

| Write Value | Effect |
|------------|--------|
| `0x00000000` | Disable IP |
| `0x00000001` | Enable IP, no start, CLKDIV=0 (6 MHz SCLK) |
| `0x00000003` | Enable + start transfer, CLKDIV=0 (only works if BUSY=0) |
| `0x00000403` | Enable + start, CLKDIV=4 (1.2 MHz SCLK) |
| `0x00000B01` | Enable, CLKDIV=11 (500 kHz SCLK), no start |
| `0x00000B03` | Enable + start, CLKDIV=11 (500 kHz SCLK) |
| `0x00FF0001` | Enable, CLKDIV=255 (~23.4 kHz SCLK) |

### SPI_STATUS Read Encoding

| Read Value | Meaning |
|-----------|---------|
| `0x00000000` | Idle — no transfer in progress, no pending completion |
| `0x00000001` | Busy — transfer in progress |
| `0x00000002` | Done — transfer complete, RXDATA valid, clear DONE before next transfer |
| `0x00000003` | Should not occur (BUSY and DONE are mutually exclusive in correct operation) |

### SCLK Frequency Quick Reference

| CLKDIV | Hex | F_SCLK (12 MHz sys) | Application |
|--------|-----|---------------------|-------------|
| 0 | 0x00 | 6.000 MHz | Fast SPI flash |
| 1 | 0x01 | 3.000 MHz | SD cards |
| 4 | 0x04 | 1.200 MHz | General-purpose sensors |
| 5 | 0x05 | 1.000 MHz | BMI160, MPU6000 |
| 11 | 0x0B | 500 kHz | DAC/ADC at moderate speed |
| 23 | 0x17 | 250 kHz | Slow sensors |
| 59 | 0x3B | 100 kHz | Very slow peripherals |
| 255 | 0xFF | ~23.4 kHz | Ultra-slow / debug |
