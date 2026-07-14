# SPI Master IP — Register Map

**Base Address:** `IO_BASE = 0x400000` | **Register Width:** 32 bits | **Access:** 32-bit word only

---

## 1. Register Summary

| Register | Symbol | Byte Offset | Word Addr | Absolute Address | Access | Reset |
|----------|--------|-------------|-----------|-----------------|--------|-------|
| Control | `SPI_CTRL` | 64 (0x40) | 16 | 0x400040 | R/W | 0x00000000 |
| Transmit Data | `SPI_TXDATA` | 68 (0x44) | 17 | 0x400044 | R/W | 0x00000000 |
| Receive Data | `SPI_RXDATA` | 72 (0x48) | 18 | 0x400048 | R | 0x00000000 |
| Status | `SPI_STATUS` | 76 (0x4C) | 19 | 0x40004C | R/W | 0x00000000 |

**io.h macros:**
```c
#define IO_BASE    0x400000
#define SPI_CTRL   64
#define SPI_TXDATA 68
#define SPI_RXDATA 72
#define SPI_STATUS 76

#define IO_IN(port)       *(volatile uint32_t*)(IO_BASE + port)
#define IO_OUT(port,val)  *(volatile uint32_t*)(IO_BASE + port)=(val)
```

---

## 2. Address Map

```
IO_BASE = 0x400000
├── 0x400004  IO_LEDS       (existing)
├── 0x400008  IO_UART_DAT   (existing)
├── 0x400010  IO_UART_CNTL  (existing)
├── 0x400020  IO_GPIO_DATA  (Task-5)
├── 0x400024  IO_GPIO_DIR   (Task-5)
├── 0x400028  IO_GPIO_READ  (Task-5)
├── 0x400040  SPI_CTRL      ← Task-7
├── 0x400044  SPI_TXDATA    ← Task-7
├── 0x400048  SPI_RXDATA    ← Task-7
└── 0x40004C  SPI_STATUS    ← Task-7
```

---

## 3. SPI_CTRL — Control Register (Offset 64)

**Absolute:** `0x400040` | **Word addr:** 16 | **Access:** R/W | **Reset:** `0x00000000`

```
Bit:  31      16  15       8   7    2   1    0
      ┌──────────┬──────────┬────────┬────┬────┐
      │ Reserved │  CLKDIV  │  Rsvd  │ ST │ EN │
      │  [31:16] │  [15:8]  │  [7:2] │[1] │[0] │
      └──────────┴──────────┴────────┴────┴────┘
```

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:16] | Reserved | R | 0 | Read as zero |
| [15:8] | `CLKDIV` | R/W | 0x00 | SPI clock divider. `F_SCLK = F_SYS / (2×(CLKDIV+1))` |
| [7:2] | Reserved | R | 0 | Read as zero |
| [1] | `START` | W (auto-clear) | 0 | Write 1 to start transfer. Fires only if EN=1 and BUSY=0. Always reads back 0 |
| [0] | `EN` | R/W | 0 | Enable the SPI IP. Must be 1 for any transfer |

**Example writes:**
```c
IO_OUT(SPI_CTRL, (4 << 8) | 0x03);   // CLKDIV=4, START=1, EN=1
IO_OUT(SPI_CTRL, (11 << 8) | 0x01);  // CLKDIV=11, EN=1, no start
IO_OUT(SPI_CTRL, 0x00);               // Disable
```

---

## 4. SPI_TXDATA — Transmit Data Register (Offset 68)

**Absolute:** `0x400044` | **Word addr:** 17 | **Access:** R/W | **Reset:** `0x00000000`

```
Bit:  31              8   7            0
      ┌───────────────────┬────────────┐
      │     Reserved      │   TXDATA   │
      │      [31:8]       │   [7:0]    │
      └───────────────────┴────────────┘
```

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:8] | Reserved | R | 0 | Read as zero; writes ignored |
| [7:0] | `TXDATA` | R/W | 0x00 | Byte to transmit. Write before asserting START. MSB (bit 7) transmitted first |

Always write `SPI_TXDATA` **before** writing `SPI_CTRL` with START=1. The IP latches `tx_data` into `shift_reg` the moment START fires.

---

## 5. SPI_RXDATA — Receive Data Register (Offset 72)

**Absolute:** `0x400048` | **Word addr:** 18 | **Access:** R (writes ignored) | **Reset:** `0x00000000`

```
Bit:  31              8   7            0
      ┌───────────────────┬────────────┐
      │     Reserved      │   RXDATA   │
      │      [31:8]       │   [7:0]    │
      └───────────────────┴────────────┘
```

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:8] | Reserved | R | 0 | Always zero |
| [7:0] | `RXDATA` | R | 0x00 | Byte received from SPI slave. Valid after `SPI_STATUS.DONE=1`. Holds value until next transfer |

Read RXDATA only after confirming DONE=1:
```c
while (!(IO_IN(SPI_STATUS) & 0x2));
uint8_t rx = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
```

---

## 6. SPI_STATUS — Status Register (Offset 76)

**Absolute:** `0x40004C` | **Word addr:** 19 | **Access:** R/W | **Reset:** `0x00000000`

```
Bit:  31               2   1     0
      ┌─────────────────────┬──────┬──────┐
      │       Reserved      │ DONE │ BUSY │
      │        [31:2]       │ [1]  │ [0]  │
      └─────────────────────┴──────┴──────┘
                               R/W1C   R
```

| Bits | Name | Access | Reset | Description |
|------|------|--------|-------|-------------|
| [31:2] | Reserved | R | 0 | Read as zero |
| [1] | `DONE` | R/W1C | 0 | Set by hardware when transfer completes. Clear by writing 1 (write 0 = no effect). RXDATA is valid when DONE=1 |
| [0] | `BUSY` | R | 0 | Set when transfer starts. Cleared when transfer completes. Read-only |

**Status state machine:**
```
IDLE       BUSY=0, DONE=0   (ready for transfer)
    │ START fires
    ▼
ACTIVE     BUSY=1, DONE=0   (transfer in progress)
    │ Transfer completes
    ▼
DONE       BUSY=0, DONE=1   (RXDATA valid)
    │ SW writes 0x2 to STATUS
    ▼
IDLE       BUSY=0, DONE=0   (ready for next)
```

**Programming examples:**
```c
while (!(IO_IN(SPI_STATUS) & 0x2));   // poll DONE
IO_OUT(SPI_STATUS, 0x2);              // clear DONE
if (IO_IN(SPI_STATUS) & 0x1) { }      // check BUSY
```

---

## 7. Register Access Rules

| Register | Write | Read |
|----------|-------|------|
| `SPI_CTRL` | Updates EN, CLKDIV; if START=1 & EN=1 & !BUSY → starts transfer | CLKDIV in [15:8], EN in [0], START always 0 |
| `SPI_TXDATA` | Stores byte in tx_data holding register | Last written value |
| `SPI_RXDATA` | **Ignored** | Last received byte, 0x00 after reset |
| `SPI_STATUS` | Bit1 (DONE): write 1 clears; all other writes ignored | Current DONE [1], BUSY [0] |

---

## 8. Reset Values Summary

| Register | Field | Reset | Notes |
|----------|-------|-------|-------|
| `SPI_CTRL` | CLKDIV | 0x04 | Default SCLK = 1.2 MHz at 12 MHz |
| `SPI_CTRL` | EN, START | 0 | Disabled at reset |
| `SPI_TXDATA` | TXDATA | 0x00 | — |
| `SPI_RXDATA` | RXDATA | 0x00 | — |
| `SPI_STATUS` | DONE, BUSY | 0 | Idle at reset |

---

## 9. Encoding Reference

### SPI_CTRL Write Examples

| Write Value | Effect |
|------------|--------|
| `0x00000000` | Disable IP |
| `0x00000001` | Enable, CLKDIV=0 (6 MHz SCLK) |
| `0x00000003` | Enable + start, CLKDIV=0 |
| `0x00000403` | Enable + start, CLKDIV=4 (1.2 MHz) |
| `0x00000B03` | Enable + start, CLKDIV=11 (500 kHz) |
| `0x00FF0001` | Enable, CLKDIV=255 (~23.4 kHz) |

### SCLK Frequency Reference

| CLKDIV | F_SCLK (12 MHz) | Typical use |
|--------|----------------|-------------|
| 0 | 6.000 MHz | SPI flash |
| 1 | 3.000 MHz | SD cards |
| 4 | 1.200 MHz | General sensors |
| 5 | 1.000 MHz | BMI160, MPU6000 |
| 11 | 500 kHz | DAC/ADC |
| 23 | 250 kHz | Slow sensors |
| 59 | 100 kHz | Very slow peripherals |
| 255 | ~23.4 kHz | Debug |
