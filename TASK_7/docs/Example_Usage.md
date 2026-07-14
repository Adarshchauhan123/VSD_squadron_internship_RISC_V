# SPI Master IP — Example Usage

**IP Name:** SPI Master  
**Version:** 1.0  
**Target:** VSDSquadron FM — FemtoRV32 RISC-V SoC  

All examples assume the `io.h` macros are defined:

```c
#include "io.h"
// SPI_CTRL   = 64  (0x400040)
// SPI_TXDATA = 68  (0x400044)
// SPI_RXDATA = 72  (0x400048)
// SPI_STATUS = 76  (0x40004C)
```

---

## Table of Contents

1. [io.h — Complete Header File](#1-ioh--complete-header-file)
2. [Core Transfer Function](#2-core-transfer-function)
3. [Example 1 — Single-Byte Loopback Test](#3-example-1--single-byte-loopback-test)
4. [Example 2 — Multi-Byte Transaction (Loop)](#4-example-2--multi-byte-transaction-loop)
5. [Example 3 — SPI Flash (W25Q16) Read Device ID](#5-example-3--spi-flash-w25q16-read-device-id)
6. [Example 4 — SPI ADC (MCP3002) Read Channel 0](#6-example-4--spi-adc-mcp3002-read-channel-0)
7. [Example 5 — SPI LED Driver (MAX7219) — 7-Segment Display](#7-example-5--spi-led-driver-max7219--7-segment-display)
8. [Example 6 — Clock Speed Selection](#8-example-6--clock-speed-selection)
9. [Example 7 — BUSY Timeout Guard](#9-example-7--busy-timeout-guard)
10. [Testbench — spi_tb.v (Complete)](#10-testbench--spi_tbv-complete)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. io.h — Complete Header File

```c
/* io.h — VSDSquadron FM SoC IO Register Definitions
 * All peripherals share IO_BASE = 0x400000
 * Registers are 32-bit word-aligned.
 */
#ifndef IO_H
#define IO_H

#include <stdint.h>

/* ── Base address ─────────────────────────────────────────────────────── */
#define IO_BASE       0x400000

/* ── Existing peripherals (Tasks 1-5) ────────────────────────────────── */
#define IO_LEDS       4     /* word addr  1 — 5-bit LED register       */
#define IO_UART_DAT   8     /* word addr  2 — UART transmit data        */
#define IO_UART_CNTL  16    /* word addr  4 — UART control/status       */
#define IO_GPIO_DATA  32    /* word addr  8 — GPIO output data          */
#define IO_GPIO_DIR   36    /* word addr  9 — GPIO direction (1=output) */
#define IO_GPIO_READ  40    /* word addr 10 — GPIO input read           */

/* ── SPI Master (Task-7) ─────────────────────────────────────────────── */
#define SPI_CTRL      64    /* word addr 16 — EN, START, CLKDIV         */
#define SPI_TXDATA    68    /* word addr 17 — Transmit byte             */
#define SPI_RXDATA    72    /* word addr 18 — Receive byte (read-only)  */
#define SPI_STATUS    76    /* word addr 19 — BUSY (r), DONE (r/w1c)   */

/* ── SCLK frequency (12 MHz system clock): F = 12MHz / (2*(CLKDIV+1)) ─ */
#define SPI_CLKDIV_6MHZ    0    /* 6.000 MHz */
#define SPI_CLKDIV_3MHZ    1    /* 3.000 MHz */
#define SPI_CLKDIV_2MHZ    2    /* 2.000 MHz */
#define SPI_CLKDIV_1MHZ    5    /* 1.000 MHz */
#define SPI_CLKDIV_500KHZ  11   /* 500 kHz   */
#define SPI_CLKDIV_250KHZ  23   /* 250 kHz   */
#define SPI_CLKDIV_100KHZ  59   /* 100 kHz   */

/* ── SPI_STATUS bit masks ─────────────────────────────────────────────── */
#define SPI_BUSY      (1u << 0)
#define SPI_DONE      (1u << 1)

/* ── SPI_CTRL bit masks ───────────────────────────────────────────────── */
#define SPI_EN        (1u << 0)
#define SPI_START     (1u << 1)

/* ── Register access macros ───────────────────────────────────────────── */
#define IO_IN(port)       (*(volatile uint32_t*)((uint32_t)IO_BASE + (port)))
#define IO_OUT(port,val)  (*(volatile uint32_t*)((uint32_t)IO_BASE + (port))) = (val)

/* ── Helper: build CTRL write value ─────────────────────────────────── */
#define SPI_CTRL_VAL(clkdiv, start, en) \
    (((uint32_t)(clkdiv) << 8) | ((start) ? SPI_START : 0) | ((en) ? SPI_EN : 0))

#endif /* IO_H */
```

---

## 2. Core Transfer Function

This is the fundamental building block used in all examples below.

```c
/* ─────────────────────────────────────────────────────────────────────────
 * spi_transfer() — Send one byte, receive one byte
 *
 * Parameters:
 *   clkdiv  — CLKDIV field value (8-bit).
 *             F_SCLK = 12MHz / (2*(clkdiv+1))
 *             Use SPI_CLKDIV_* macros from io.h.
 *   tx      — byte to transmit (MSB first)
 *
 * Returns:
 *   byte received from slave during this transfer
 *
 * Side effects:
 *   CS_N is asserted (low) during transfer and de-asserted (high) after.
 *   DONE flag is cleared before return.
 * ─────────────────────────────────────────────────────────────────────── */
static uint8_t spi_transfer(uint8_t clkdiv, uint8_t tx) {
    /* 1. Load TX byte */
    IO_OUT(SPI_TXDATA, (uint32_t)tx);

    /* 2. Start transfer: CLKDIV | START=1 | EN=1 */
    IO_OUT(SPI_CTRL, ((uint32_t)clkdiv << 8) | SPI_START | SPI_EN);

    /* 3. Poll DONE flag */
    while (!(IO_IN(SPI_STATUS) & SPI_DONE));

    /* 4. Clear DONE (write-1-to-clear) */
    IO_OUT(SPI_STATUS, SPI_DONE);

    /* 5. Return received byte */
    return (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
}
```

---

## 3. Example 1 — Single-Byte Loopback Test

Tests the SPI IP in isolation using a wire loopback (MOSI pin 9 connected to MISO pin 10 externally, or using the `ifdef BENCH` internal loopback for simulation).

```c
/* ─── Example 1: Single-byte loopback test ─────────────────────────────
 * Hardware: Connect MOSI (pin 9) to MISO (pin 10) with a jumper wire.
 * Expected: RXDATA == TXDATA for each test byte.
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"

static void print_hex(uint32_t v) {
    int i;
    for (i = 7; i >= 0; i--) {
        uint8_t n = (v >> (i * 4)) & 0xF;
        while (IO_IN(IO_UART_CNTL) & 0x100);
        IO_OUT(IO_UART_DAT, (n < 10) ? ('0' + n) : ('A' + n - 10));
    }
}

static void print_char(char c) {
    while (IO_IN(IO_UART_CNTL) & 0x100);
    IO_OUT(IO_UART_DAT, c);
}

static void print_str(const char *s) {
    while (*s) {
        while (IO_IN(IO_UART_CNTL) & 0x100);
        IO_OUT(IO_UART_DAT, *s++);
    }
}

/* Returns 1 if PASS, 0 if FAIL */
static int test_loopback(uint8_t clkdiv, uint8_t tx_byte) {
    uint8_t rx = spi_transfer(clkdiv, tx_byte);
    print_str("TX=0x");
    print_hex(tx_byte);
    print_str(" RX=0x");
    print_hex(rx);
    if (rx == tx_byte) {
        print_str(" PASS\n");
        return 1;
    } else {
        print_str(" FAIL\n");
        return 0;
    }
}

void main() {
    int pass_count = 0;
    int total = 0;

    print_str("=== SPI Loopback Test ===\n");

    /* Test at different speeds */
    pass_count += test_loopback(SPI_CLKDIV_1MHZ,   0xA5); total++;
    pass_count += test_loopback(SPI_CLKDIV_500KHZ, 0x3C); total++;
    pass_count += test_loopback(SPI_CLKDIV_250KHZ, 0xFF); total++;
    pass_count += test_loopback(SPI_CLKDIV_100KHZ, 0x00); total++;
    pass_count += test_loopback(SPI_CLKDIV_1MHZ,   0xAA); total++;
    pass_count += test_loopback(SPI_CLKDIV_1MHZ,   0x55); total++;

    print_str("\nResult: ");
    print_hex(pass_count);
    print_str("/");
    print_hex(total);
    print_str(" PASS\n");
}
```

### Expected Output (via serial terminal at 9600 baud)

```
=== SPI Loopback Test ===
TX=0x000000A5 RX=0x000000A5 PASS
TX=0x0000003C RX=0x0000003C PASS
TX=0x000000FF RX=0x000000FF PASS
TX=0x00000000 RX=0x00000000 PASS
TX=0x000000AA RX=0x000000AA PASS
TX=0x00000055 RX=0x00000055 PASS

Result: 0x00000006/0x00000006 PASS
```

---

## 4. Example 2 — Multi-Byte Transaction (Loop)

Sends a buffer of bytes sequentially. Each byte is a separate 8-bit SPI transfer (CS_N de-asserts between bytes since the IP is single-byte only).

```c
/* ─── Example 2: Multi-byte sequential transfers ────────────────────────
 * Note: CS_N is automatically controlled per-byte by the IP.
 * For devices requiring CS_N held low across multiple bytes, use GPIO
 * to drive an additional CS pin and leave the IP's CS_N disconnected.
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"
#include <stdint.h>

#define TX_LEN 4

static const uint8_t tx_buf[TX_LEN] = {0x01, 0x02, 0x04, 0x08};
static uint8_t rx_buf[TX_LEN];

void main() {
    int i;

    /* Send all bytes at 500 kHz */
    for (i = 0; i < TX_LEN; i++) {
        rx_buf[i] = spi_transfer(SPI_CLKDIV_500KHZ, tx_buf[i]);
    }

    /* Print results */
    for (i = 0; i < TX_LEN; i++) {
        print_str("TX[");
        print_hex(i);
        print_str("]=0x");
        print_hex(tx_buf[i]);
        print_str("  RX[");
        print_hex(i);
        print_str("]=0x");
        print_hex(rx_buf[i]);
        print_str("\n");
    }
}
```

---

## 5. Example 3 — SPI Flash (W25Q16) Read Device ID

Reads the Manufacturer ID and Device ID from a W25Qxx SPI NOR Flash.

**SPI Flash command protocol:**  
- Send `0x9F` (JEDEC Read ID)  
- Receive 3 bytes: [Manufacturer ID, Memory Type, Capacity]

```c
/* ─── Example 3: W25Q16 SPI Flash — Read JEDEC Device ID ───────────────
 * Command: 0x9F (JEDEC Read ID)
 * Response: 3 bytes — [Mfr ID][Mem Type][Capacity]
 * W25Q16 expected: 0xEF 0x40 0x15
 *
 * Hardware connections:
 *   FPGA SCLK (pin 6) → Flash CLK
 *   FPGA MOSI (pin 9) → Flash DI
 *   FPGA MISO (pin 10)← Flash DO
 *   FPGA CS_N (pin 11)→ Flash /CS
 *
 * Limitation: CS_N de-asserts between each byte. Some flash devices
 * require CS_N held low for all bytes of a command. In that case,
 * use a GPIO pin as CS and modify the integration accordingly.
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"

void main() {
    uint8_t mfr_id, mem_type, capacity;

    /* JEDEC Read ID: 0x9F command, then 3 RX bytes */
    /* Note: Because CS_N de-asserts between bytes, this only works
     * with devices that accept repeated single-byte CS sequences,
     * or if you use a GPIO for CS and tie IP CS_N to VCC.
     * This example demonstrates the transfer API; verify against
     * your specific flash device's timing diagram. */

    spi_transfer(SPI_CLKDIV_1MHZ, 0x9F);  /* Send command */
    mfr_id    = spi_transfer(SPI_CLKDIV_1MHZ, 0x00);  /* Dummy, read MFR ID */
    mem_type  = spi_transfer(SPI_CLKDIV_1MHZ, 0x00);  /* Dummy, read Mem Type */
    capacity  = spi_transfer(SPI_CLKDIV_1MHZ, 0x00);  /* Dummy, read Capacity */

    print_str("Flash ID: 0x");
    print_hex(mfr_id);
    print_str(" 0x");
    print_hex(mem_type);
    print_str(" 0x");
    print_hex(capacity);
    print_str("\n");

    /* W25Q16 expected: EF 40 15 */
    if (mfr_id == 0xEF && mem_type == 0x40) {
        print_str("Winbond W25Qxx detected\n");
    }
}
```

---

## 6. Example 4 — SPI ADC (MCP3002) Read Channel 0

Reads a 10-bit ADC sample from channel 0 of a Microchip MCP3002 2-channel SPI ADC.

```c
/* ─── Example 4: MCP3002 SPI ADC — Read Channel 0 ──────────────────────
 * MCP3002 protocol (SPI Mode 0, 8-bit words):
 *   Byte 1: 0x68 = Start(1) | SGL(1) | ODD/SIGN(0) | MSBF(1) | padding(000)
 *   Byte 2: 0x00 = dummy, receives upper 2 bits of 10-bit result
 *   Result = ((byte2[1:0] << 8) | byte3) — Note: MCP3002 uses 2 SPI bytes
 *
 * Hardware:
 *   VDD  → 3.3V
 *   DGND → GND
 *   AGND → GND
 *   CH0  → analog input signal
 *   CLK  ← FPGA SCLK (pin 6)
 *   Dout → FPGA MISO (pin 10)
 *   Din  ← FPGA MOSI (pin 9)
 *   /CS  ← FPGA CS_N (pin 11)
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"

/* Read MCP3002 Channel 0, single-ended
 * Returns: 10-bit ADC value (0–1023) */
static uint16_t mcp3002_read_ch0(void) {
    uint8_t b1, b2;

    /* Send start + config byte, receive MSBs */
    b1 = spi_transfer(SPI_CLKDIV_500KHZ, 0x68);
    /* Send dummy byte, receive LSBs */
    b2 = spi_transfer(SPI_CLKDIV_500KHZ, 0x00);

    /* Reconstruct 10-bit result */
    return (uint16_t)(((b1 & 0x03) << 8) | b2);
}

void main() {
    uint16_t adc_val;
    int i;

    print_str("MCP3002 ADC Readings:\n");

    for (i = 0; i < 8; i++) {
        adc_val = mcp3002_read_ch0();
        print_str("Sample[");
        print_hex(i);
        print_str("]: ");
        print_hex(adc_val);   /* 0x000-0x3FF */
        print_str("\n");

        /* Simple delay between samples */
        volatile int d;
        for (d = 0; d < 10000; d++);
    }
}
```

---

## 7. Example 5 — SPI LED Driver (MAX7219) — 7-Segment Display

Sends commands to a MAX7219 8-digit LED controller. The MAX7219 uses 16-bit transfers (two bytes).

```c
/* ─── Example 5: MAX7219 7-Segment LED Driver ───────────────────────────
 * MAX7219 protocol: 16-bit frames (2 bytes per command)
 *   Byte 1: Register address (8 bits)
 *   Byte 2: Data (8 bits)
 *
 * Note: MAX7219 requires CS_N held low for both bytes of a 16-bit frame.
 * Since this IP de-asserts CS_N between bytes, the MAX7219 may latch
 * data only on the first byte. For proper operation:
 *   Option A: Use a GPIO pin as CS and tie IP CS_N to VCC.
 *   Option B: Use this as a wiring example and adjust for your hardware.
 *
 * This example demonstrates the software pattern; hardware CS modification
 * may be needed for strict MAX7219 timing compliance.
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"

/* MAX7219 register addresses */
#define MAX7219_REG_NOOP      0x00
#define MAX7219_REG_DIGIT(n)  (0x01 + (n))   /* Digits 0-7 */
#define MAX7219_REG_DECODE    0x09
#define MAX7219_REG_INTENSITY 0x0A
#define MAX7219_REG_SCANLIMIT 0x0B
#define MAX7219_REG_SHUTDOWN  0x0C
#define MAX7219_REG_TEST      0x0F

/* Send 16-bit frame to MAX7219 */
static void max7219_write(uint8_t reg, uint8_t data) {
    spi_transfer(SPI_CLKDIV_1MHZ, reg);   /* Address byte */
    spi_transfer(SPI_CLKDIV_1MHZ, data);  /* Data byte    */
}

/* Initialize MAX7219 */
static void max7219_init(void) {
    max7219_write(MAX7219_REG_SHUTDOWN,  0x01); /* Normal operation (not shutdown) */
    max7219_write(MAX7219_REG_DECODE,    0xFF); /* BCD decode for all digits */
    max7219_write(MAX7219_REG_INTENSITY, 0x08); /* Half brightness */
    max7219_write(MAX7219_REG_SCANLIMIT, 0x07); /* All 8 digits active */
    max7219_write(MAX7219_REG_TEST,      0x00); /* No display test */
}

/* Display a decimal number (0–99999999) on 8-digit display */
static void max7219_display(uint32_t value) {
    int i;
    uint8_t digits[8];

    for (i = 0; i < 8; i++) {
        digits[i] = value % 10;
        value /= 10;
    }

    for (i = 0; i < 8; i++) {
        max7219_write(MAX7219_REG_DIGIT(i), digits[i]);
    }
}

void main() {
    max7219_init();

    /* Display "12345678" */
    max7219_display(12345678);
    print_str("MAX7219: Displaying 12345678\n");

    /* Count up */
    uint32_t counter = 0;
    while (1) {
        max7219_display(counter++);
        volatile int d;
        for (d = 0; d < 50000; d++);
        if (counter > 99999999) counter = 0;
    }
}
```

---

## 8. Example 6 — Clock Speed Selection

Demonstrates how to select the correct CLKDIV for a target SCLK frequency.

```c
/* ─── Example 6: Clock speed selection ─────────────────────────────────
 * Selects CLKDIV based on target frequency at 12 MHz system clock.
 * Formula: CLKDIV = (F_SYS / (2 * F_SCLK)) - 1
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"

/* CLKDIV for common target frequencies at 12 MHz system clock */
typedef struct {
    uint32_t   target_hz;
    uint8_t    clkdiv;
    const char *label;
} spi_speed_t;

static const spi_speed_t speeds[] = {
    { 6000000, 0,  "6.000 MHz" },
    { 3000000, 1,  "3.000 MHz" },
    { 2000000, 2,  "2.000 MHz" },
    { 1200000, 4,  "1.200 MHz" },
    { 1000000, 5,  "1.000 MHz" },
    {  500000, 11, " 500 kHz " },
    {  250000, 23, " 250 kHz " },
    {  100000, 59, " 100 kHz " },
    {   50000, 119," 50 kHz  " },
};
#define NUM_SPEEDS (sizeof(speeds) / sizeof(speeds[0]))

void main() {
    uint8_t rx;
    int i;

    print_str("SPI Speed Test (loopback — connect MOSI to MISO)\n");
    print_str("Speed       | TX     | RX     | Result\n");
    print_str("------------|--------|--------|-------\n");

    for (i = 0; i < (int)NUM_SPEEDS; i++) {
        rx = spi_transfer(speeds[i].clkdiv, 0xAB);
        print_str(speeds[i].label);
        print_str(" | 0x");
        print_hex(0xAB);
        print_str(" | 0x");
        print_hex(rx);
        print_str(" | ");
        print_str((rx == 0xAB) ? "PASS\n" : "FAIL\n");
    }
}
```

---

## 9. Example 7 — BUSY Timeout Guard

In production code, polling without a timeout can cause the system to hang if the SPI hardware malfunctions. This example adds a timeout counter.

```c
/* ─── Example 7: Transfer with timeout ─────────────────────────────────
 * If DONE is not set within TIMEOUT_CYCLES, return 0xFF as error value.
 * TIMEOUT_CYCLES should be >> 16 * (CLKDIV+1) * margin
 * At CLKDIV=4, one transfer = 80 cycles; timeout at 10000 is conservative.
 * ─────────────────────────────────────────────────────────────────────── */
#include "io.h"
#include <stdint.h>

#define SPI_TIMEOUT_CYCLES 100000u

typedef enum {
    SPI_OK      = 0,
    SPI_TIMEOUT = 1,
    SPI_BUSY    = 2
} spi_status_t;

/* Transfer with timeout.
 * Returns SPI_OK on success, SPI_TIMEOUT if DONE not seen in time.
 * rx_out: pointer to store received byte (set to 0xFF on error). */
static spi_status_t spi_transfer_safe(uint8_t clkdiv, uint8_t tx,
                                       uint8_t *rx_out) {
    uint32_t timeout;

    /* Check if a previous transfer is still running */
    if (IO_IN(SPI_STATUS) & SPI_BUSY) {
        *rx_out = 0xFF;
        return SPI_BUSY;
    }

    IO_OUT(SPI_TXDATA, (uint32_t)tx);
    IO_OUT(SPI_CTRL, ((uint32_t)clkdiv << 8) | SPI_START | SPI_EN);

    for (timeout = 0; timeout < SPI_TIMEOUT_CYCLES; timeout++) {
        if (IO_IN(SPI_STATUS) & SPI_DONE) {
            IO_OUT(SPI_STATUS, SPI_DONE);        /* clear DONE */
            *rx_out = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
            return SPI_OK;
        }
    }

    /* Timeout — reset the IP to recover */
    IO_OUT(SPI_CTRL, 0x00);   /* disable EN (no transfer can start) */
    *rx_out = 0xFF;
    return SPI_TIMEOUT;
}

void main() {
    uint8_t rx;
    spi_status_t result;

    result = spi_transfer_safe(SPI_CLKDIV_1MHZ, 0xA5, &rx);

    if (result == SPI_OK) {
        print_str("Transfer OK, RX=0x");
        print_hex(rx);
        print_str("\n");
    } else if (result == SPI_TIMEOUT) {
        print_str("ERROR: SPI transfer timed out\n");
    } else if (result == SPI_BUSY) {
        print_str("ERROR: SPI peripheral busy\n");
    }
}
```

---

## 10. Testbench — spi_tb.v (Complete)

Ready-to-run Verilog testbench. Uses MOSI→MISO loopback (`assign MISO = MOSI`).

```verilog
`timescale 1ns/1ps

module spi_tb;
    // ── DUT connections ─────────────────────────────────────────────────
    reg         clk, resetn;
    reg         isIO, mem_wstrb, mem_rstrb;
    reg  [29:0] mem_wordaddr;
    reg  [31:0] mem_wdata;
    wire [31:0] spi_rdata;
    wire        spi_sel;
    wire        SCLK, MOSI, CS_N;
    wire        MISO;

    // ── MOSI→MISO loopback ───────────────────────────────────────────────
    assign MISO = MOSI;

    // ── DUT instantiation ────────────────────────────────────────────────
    spi_master DUT (
        .clk         (clk        ),
        .resetn      (resetn     ),
        .isIO        (isIO       ),
        .mem_wstrb   (mem_wstrb  ),
        .mem_rstrb   (mem_rstrb  ),
        .mem_wordaddr(mem_wordaddr),
        .mem_wdata   (mem_wdata  ),
        .spi_rdata   (spi_rdata  ),
        .spi_sel     (spi_sel    ),
        .SCLK        (SCLK       ),
        .MOSI        (MOSI       ),
        .CS_N        (CS_N       ),
        .MISO        (MISO       )
    );

    // ── Clock: 100 MHz (10 ns period) ────────────────────────────────────
    initial clk = 0;
    always  #5 clk = ~clk;

    // ── Task: write a register ───────────────────────────────────────────
    task write_reg;
        input [29:0] addr;
        input [31:0] data;
        begin
            @(posedge clk); #1;
            isIO         = 1;
            mem_wstrb    = 1;
            mem_wordaddr = addr;
            mem_wdata    = data;
            @(posedge clk); #1;
            mem_wstrb = 0;
            isIO      = 0;
        end
    endtask

    // ── Task: read a register ────────────────────────────────────────────
    task read_reg;
        input  [29:0] addr;
        output [31:0] val;
        begin
            @(posedge clk); #1;
            isIO         = 1;
            mem_rstrb    = 1;
            mem_wordaddr = addr;
            @(posedge clk); #1;
            val       = spi_rdata;
            mem_rstrb = 0;
            isIO      = 0;
        end
    endtask

    // ── Task: perform one SPI transfer and return RX value ───────────────
    task spi_do_transfer;
        input  [7:0]  tx_byte;
        input  [7:0]  clkdiv;
        output [31:0] rx_val;
        integer       poll_count;
        reg    [31:0] status_val;
        begin
            poll_count = 0;
            // Write TXDATA
            write_reg(30'd17, {24'h0, tx_byte});
            $display("  TX byte: 0x%02X", tx_byte);
            // Write CTRL: CLKDIV, START=1, EN=1
            write_reg(30'd16, {16'h0, clkdiv, 6'h0, 1'b1, 1'b1});
            // Poll STATUS.DONE
            begin : poll_loop
                forever begin
                    read_reg(30'd19, status_val);
                    poll_count = poll_count + 1;
                    if (status_val[1]) disable poll_loop;
                    if (poll_count > 100000) begin
                        $display("ERROR: Timeout waiting for DONE");
                        disable poll_loop;
                    end
                end
            end
            $display("  DONE after %0d polls", poll_count);
            // Read RXDATA
            read_reg(30'd18, rx_val);
            $display("  RX byte: 0x%08X", rx_val);
            // Clear DONE
            write_reg(30'd19, 32'h2);
        end
    endtask

    // ── Test sequence ────────────────────────────────────────────────────
    integer i;
    reg [31:0] rx_val, status_val;
    reg [7:0]  test_bytes [0:5];

    initial begin
        $dumpfile("spi_tb.vcd");
        $dumpvars(0, spi_tb);

        // Initialize inputs
        resetn       = 0;
        isIO         = 0;
        mem_wstrb    = 0;
        mem_rstrb    = 0;
        mem_wordaddr = 30'd0;
        mem_wdata    = 32'h0;

        // Assert reset for 5 cycles
        repeat (5) @(posedge clk);
        resetn = 1;
        repeat (2) @(posedge clk);

        // Test patterns
        test_bytes[0] = 8'hA5;
        test_bytes[1] = 8'h3C;
        test_bytes[2] = 8'hFF;
        test_bytes[3] = 8'h00;
        test_bytes[4] = 8'hAA;
        test_bytes[5] = 8'h55;

        $display("=== SPI Master Testbench ===");
        $display("MOSI→MISO loopback active");
        $display("");

        for (i = 0; i < 6; i = i + 1) begin
            $display("--- Transfer %0d: TX=0x%02X ---", i, test_bytes[i]);
            spi_do_transfer(test_bytes[i], 8'd4, rx_val);
            if (rx_val[7:0] == test_bytes[i]) begin
                $display("  RESULT: PASS (RX=0x%02X)", rx_val[7:0]);
            end else begin
                $display("  RESULT: FAIL (expected 0x%02X, got 0x%02X)",
                         test_bytes[i], rx_val[7:0]);
            end
            $display("");

            // Verify STATUS is cleared
            read_reg(30'd19, status_val);
            if (status_val != 32'h0) begin
                $display("  WARNING: STATUS not fully cleared: 0x%08X", status_val);
            end
        end

        $display("=== Testbench Complete ===");
        #100;
        $finish;
    end

    // ── Waveform monitors ────────────────────────────────────────────────
    initial begin
        $monitor("t=%0t CS_N=%b SCLK=%b MOSI=%b MISO=%b spi_sel=%b",
                 $time, CS_N, SCLK, MOSI, MISO, spi_sel);
    end

endmodule
```

### Compile and Run

```bash
iverilog -o spi_tb spi_tb.v spi_master.v
vvp spi_tb
gtkwave spi_tb.vcd &
```

### Expected Output

```
=== SPI Master Testbench ===
MOSI→MISO loopback active

--- Transfer 0: TX=0xA5 ---
  TX byte: 0xA5
  DONE after 19 polls
  RX byte: 0x000000A5
  RESULT: PASS (RX=0xA5)

--- Transfer 1: TX=0x3C ---
  TX byte: 0x3C
  DONE after 19 polls
  RX byte: 0x0000003C
  RESULT: PASS (RX=0x3C)

--- Transfer 2: TX=0xFF ---
  TX byte: 0xFF
  DONE after 19 polls
  RX byte: 0x000000FF
  RESULT: PASS (RX=0xFF)

--- Transfer 3: TX=0x00 ---
  TX byte: 0x00
  DONE after 19 polls
  RX byte: 0x00000000
  RESULT: PASS (RX=0x00)

--- Transfer 4: TX=0xAA ---
  TX byte: 0xAA
  DONE after 19 polls
  RX byte: 0x000000AA
  RESULT: PASS (RX=0xAA)

--- Transfer 5: TX=0x55 ---
  TX byte: 0x55
  DONE after 19 polls
  RX byte: 0x00000055
  RESULT: PASS (RX=0x55)

=== Testbench Complete ===
```

---

## 11. Quick Reference Cheat Sheet

### Minimum Transfer Sequence (3 writes, 1 poll, 1 read, 1 write)

```c
IO_OUT(SPI_TXDATA, tx);                             // 1. load TX byte
IO_OUT(SPI_CTRL, (clkdiv << 8) | 0x3);             // 2. start (EN=1, START=1)
while (!(IO_IN(SPI_STATUS) & 0x2));                 // 3. poll DONE
IO_OUT(SPI_STATUS, 0x2);                            // 4. clear DONE
uint8_t rx = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);  // 5. read result
```

### CLKDIV Quick Reference (12 MHz system clock)

| Target SCLK | CLKDIV | Macro |
|-------------|--------|-------|
| 6 MHz | 0 | `SPI_CLKDIV_6MHZ` |
| 3 MHz | 1 | — |
| 1.2 MHz | 4 | — |
| 1 MHz | 5 | `SPI_CLKDIV_1MHZ` |
| 500 kHz | 11 | `SPI_CLKDIV_500KHZ` |
| 250 kHz | 23 | `SPI_CLKDIV_250KHZ` |
| 100 kHz | 59 | `SPI_CLKDIV_100KHZ` |

### SPI_CTRL Write Quick Reference

| Goal | Write Value |
|------|------------|
| Disable IP | `0x00000000` |
| Enable, no start, CLKDIV=4 | `0x00000401` |
| Enable + start, CLKDIV=4 | `0x00000403` |
| Enable + start, CLKDIV=11 | `0x00000B03` |

### Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Writing CTRL before TXDATA | Old/garbage byte transmitted | Always write `SPI_TXDATA` first |
| Not clearing DONE before next transfer | DONE appears to re-assert immediately | Write `0x2` to `SPI_STATUS` after each transfer |
| Not waiting for BUSY=0 between transfers | Transfer immediately returns stale data | Poll `!(SPI_STATUS & BUSY)` or ensure DONE was seen (DONE implies BUSY=0) |
| Using incorrect byte offset | Wrong register accessed | Offsets are 64/68/72/76, not word addresses 16/17/18/19 |
