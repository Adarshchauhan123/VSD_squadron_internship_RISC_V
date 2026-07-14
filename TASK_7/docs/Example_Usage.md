# SPI Master IP — Example Usage

**IP Name:** SPI Master | **Version:** 1.0 | **Target:** VSDSquadron FM — FemtoRV32 RISC-V SoC

---

## Table of Contents

1. [io.h — Complete Header File](#1-ioh--complete-header-file)
2. [Core Transfer Function](#2-core-transfer-function)
3. [Example 1 — Single-Byte Loopback Test](#3-example-1--single-byte-loopback-test)
4. [Example 2 — Multi-Byte Transaction (Loop)](#4-example-2--multi-byte-transaction-loop)
5. [Example 3 — SPI Flash (W25Q16) Read Device ID](#5-example-3--spi-flash-w25q16-read-device-id)
6. [Example 4 — SPI ADC (MCP3002) Read Channel 0](#6-example-4--spi-adc-mcp3002-read-channel-0)
7. [Example 5 — SPI LED Driver (MAX7219)](#7-example-5--spi-led-driver-max7219)
8. [Example 6 — Clock Speed Selection](#8-example-6--clock-speed-selection)
9. [Example 7 — Transfer with Timeout Guard](#9-example-7--transfer-with-timeout-guard)
10. [Testbench — spi_tb.v (Complete)](#10-testbench--spi_tbv-complete)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. io.h — Complete Header File

```c
#ifndef IO_H
#define IO_H

#include <stdint.h>

#define IO_BASE       0x400000

#define IO_LEDS       4
#define IO_UART_DAT   8
#define IO_UART_CNTL  16
#define IO_GPIO_DATA  32
#define IO_GPIO_DIR   36
#define IO_GPIO_READ  40
#define SPI_CTRL      64
#define SPI_TXDATA    68
#define SPI_RXDATA    72
#define SPI_STATUS    76

#define SPI_CLKDIV_6MHZ    0
#define SPI_CLKDIV_3MHZ    1
#define SPI_CLKDIV_2MHZ    2
#define SPI_CLKDIV_1MHZ    5
#define SPI_CLKDIV_500KHZ  11
#define SPI_CLKDIV_250KHZ  23
#define SPI_CLKDIV_100KHZ  59

#define SPI_BUSY      (1u << 0)
#define SPI_DONE      (1u << 1)
#define SPI_EN        (1u << 0)
#define SPI_START     (1u << 1)

#define IO_IN(port)       (*(volatile uint32_t*)((uint32_t)IO_BASE + (port)))
#define IO_OUT(port,val)  (*(volatile uint32_t*)((uint32_t)IO_BASE + (port))) = (val)

#endif
```

---

## 2. Core Transfer Function

```c
static uint8_t spi_transfer(uint8_t clkdiv, uint8_t tx) {
    IO_OUT(SPI_TXDATA, (uint32_t)tx);
    IO_OUT(SPI_CTRL, ((uint32_t)clkdiv << 8) | SPI_START | SPI_EN);
    while (!(IO_IN(SPI_STATUS) & SPI_DONE));
    IO_OUT(SPI_STATUS, SPI_DONE);
    return (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
}
```

---

## 3. Example 1 — Single-Byte Loopback Test

Connect MOSI (pin 9) to MISO (pin 10) with a jumper wire. Expected: RXDATA == TXDATA.

```c
#include "io.h"

static void print_hex(uint32_t v) {
    int i;
    for (i = 7; i >= 0; i--) {
        uint8_t n = (v >> (i * 4)) & 0xF;
        while (IO_IN(IO_UART_CNTL) & 0x100);
        IO_OUT(IO_UART_DAT, (n < 10) ? ('0' + n) : ('A' + n - 10));
    }
}

static void print_str(const char *s) {
    while (*s) {
        while (IO_IN(IO_UART_CNTL) & 0x100);
        IO_OUT(IO_UART_DAT, *s++);
    }
}

static int test_loopback(uint8_t clkdiv, uint8_t tx_byte) {
    uint8_t rx = spi_transfer(clkdiv, tx_byte);
    print_str("TX=0x"); print_hex(tx_byte);
    print_str(" RX=0x"); print_hex(rx);
    if (rx == tx_byte) { print_str(" PASS\n"); return 1; }
    else               { print_str(" FAIL\n"); return 0; }
}

void main() {
    int pass_count = 0, total = 0;
    print_str("=== SPI Loopback Test ===\n");
    pass_count += test_loopback(SPI_CLKDIV_1MHZ,   0xA5); total++;
    pass_count += test_loopback(SPI_CLKDIV_500KHZ, 0x3C); total++;
    pass_count += test_loopback(SPI_CLKDIV_250KHZ, 0xFF); total++;
    pass_count += test_loopback(SPI_CLKDIV_100KHZ, 0x00); total++;
    pass_count += test_loopback(SPI_CLKDIV_1MHZ,   0xAA); total++;
    pass_count += test_loopback(SPI_CLKDIV_1MHZ,   0x55); total++;
    print_str("\nResult: "); print_hex(pass_count);
    print_str("/"); print_hex(total); print_str(" PASS\n");
}
```

**Expected output:**
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

Sends a buffer sequentially. CS_N de-asserts between bytes (single-byte IP limitation).

```c
#include "io.h"
#include <stdint.h>

#define TX_LEN 4

static const uint8_t tx_buf[TX_LEN] = {0x01, 0x02, 0x04, 0x08};
static uint8_t rx_buf[TX_LEN];

void main() {
    int i;
    for (i = 0; i < TX_LEN; i++) {
        rx_buf[i] = spi_transfer(SPI_CLKDIV_500KHZ, tx_buf[i]);
    }
    for (i = 0; i < TX_LEN; i++) {
        print_str("TX["); print_hex(i); print_str("]=0x"); print_hex(tx_buf[i]);
        print_str("  RX["); print_hex(i); print_str("]=0x"); print_hex(rx_buf[i]);
        print_str("\n");
    }
}
```

---

## 5. Example 3 — SPI Flash (W25Q16) Read Device ID

JEDEC Read ID command (0x9F) returns 3 bytes: Manufacturer ID, Memory Type, Capacity.  
W25Q16 expected: `0xEF 0x40 0x15`.

```c
#include "io.h"

void main() {
    uint8_t mfr_id, mem_type, capacity;

    spi_transfer(SPI_CLKDIV_1MHZ, 0x9F);
    mfr_id   = spi_transfer(SPI_CLKDIV_1MHZ, 0x00);
    mem_type = spi_transfer(SPI_CLKDIV_1MHZ, 0x00);
    capacity = spi_transfer(SPI_CLKDIV_1MHZ, 0x00);

    print_str("Flash ID: 0x"); print_hex(mfr_id);
    print_str(" 0x"); print_hex(mem_type);
    print_str(" 0x"); print_hex(capacity); print_str("\n");

    if (mfr_id == 0xEF && mem_type == 0x40)
        print_str("Winbond W25Qxx detected\n");
}
```

> **Note:** Because CS_N de-asserts between bytes, verify your flash device accepts repeated single-byte CS sequences, or use a GPIO pin as CS and tie the IP's CS_N to VCC.

---

## 6. Example 4 — SPI ADC (MCP3002) Read Channel 0

MCP3002: 2-channel 10-bit SPI ADC. Protocol: 2-byte exchange.

```c
#include "io.h"

static uint16_t mcp3002_read_ch0(void) {
    uint8_t b1, b2;
    b1 = spi_transfer(SPI_CLKDIV_500KHZ, 0x68);
    b2 = spi_transfer(SPI_CLKDIV_500KHZ, 0x00);
    return (uint16_t)(((b1 & 0x03) << 8) | b2);
}

void main() {
    uint16_t adc_val;
    int i;
    print_str("MCP3002 ADC Readings:\n");
    for (i = 0; i < 8; i++) {
        adc_val = mcp3002_read_ch0();
        print_str("Sample["); print_hex(i); print_str("]: ");
        print_hex(adc_val); print_str("\n");
        volatile int d; for (d = 0; d < 10000; d++);
    }
}
```

---

## 7. Example 5 — SPI LED Driver (MAX7219)

MAX7219 8-digit LED controller. Protocol: 16-bit frames (2 bytes: address + data).

```c
#include "io.h"

#define MAX7219_DECODE    0x09
#define MAX7219_INTENSITY 0x0A
#define MAX7219_SCANLIMIT 0x0B
#define MAX7219_SHUTDOWN  0x0C
#define MAX7219_TEST      0x0F

static void max7219_write(uint8_t reg, uint8_t data) {
    spi_transfer(SPI_CLKDIV_1MHZ, reg);
    spi_transfer(SPI_CLKDIV_1MHZ, data);
}

static void max7219_init(void) {
    max7219_write(MAX7219_SHUTDOWN,  0x01);
    max7219_write(MAX7219_DECODE,    0xFF);
    max7219_write(MAX7219_INTENSITY, 0x08);
    max7219_write(MAX7219_SCANLIMIT, 0x07);
    max7219_write(MAX7219_TEST,      0x00);
}

static void max7219_display(uint32_t value) {
    int i;
    for (i = 1; i <= 8; i++) {
        max7219_write(i, value % 10);
        value /= 10;
    }
}

void main() {
    max7219_init();
    max7219_display(12345678);
    print_str("MAX7219: Displaying 12345678\n");
}
```

---

## 8. Example 6 — Clock Speed Selection

```c
#include "io.h"

typedef struct { uint8_t clkdiv; const char *label; } spi_speed_t;

static const spi_speed_t speeds[] = {
    {  0, "6.000 MHz" },
    {  1, "3.000 MHz" },
    {  4, "1.200 MHz" },
    {  5, "1.000 MHz" },
    { 11, " 500 kHz " },
    { 23, " 250 kHz " },
    { 59, " 100 kHz " },
};
#define NUM_SPEEDS 7

void main() {
    uint8_t rx;
    int i;
    print_str("SPI Speed Test (loopback)\n");
    for (i = 0; i < NUM_SPEEDS; i++) {
        rx = spi_transfer(speeds[i].clkdiv, 0xAB);
        print_str(speeds[i].label);
        print_str(" TX=0xAB RX=0x"); print_hex(rx);
        print_str(rx == 0xAB ? " PASS\n" : " FAIL\n");
    }
}
```

---

## 9. Example 7 — Transfer with Timeout Guard

```c
#include "io.h"
#include <stdint.h>

#define SPI_TIMEOUT 100000u

typedef enum { SPI_OK = 0, SPI_TIMEOUT_ERR = 1, SPI_BUSY_ERR = 2 } spi_status_t;

static spi_status_t spi_transfer_safe(uint8_t clkdiv, uint8_t tx, uint8_t *rx_out) {
    uint32_t timeout;

    if (IO_IN(SPI_STATUS) & SPI_BUSY) { *rx_out = 0xFF; return SPI_BUSY_ERR; }

    IO_OUT(SPI_TXDATA, (uint32_t)tx);
    IO_OUT(SPI_CTRL, ((uint32_t)clkdiv << 8) | SPI_START | SPI_EN);

    for (timeout = 0; timeout < SPI_TIMEOUT; timeout++) {
        if (IO_IN(SPI_STATUS) & SPI_DONE) {
            IO_OUT(SPI_STATUS, SPI_DONE);
            *rx_out = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
            return SPI_OK;
        }
    }

    IO_OUT(SPI_CTRL, 0x00);
    *rx_out = 0xFF;
    return SPI_TIMEOUT_ERR;
}

void main() {
    uint8_t rx;
    spi_status_t result = spi_transfer_safe(SPI_CLKDIV_1MHZ, 0xA5, &rx);

    if      (result == SPI_OK)          { print_str("OK RX=0x"); print_hex(rx); print_str("\n"); }
    else if (result == SPI_TIMEOUT_ERR) { print_str("ERROR: timeout\n"); }
    else if (result == SPI_BUSY_ERR)    { print_str("ERROR: busy\n"); }
}
```

---

## 10. Testbench — spi_tb.v (Complete)

MOSI→MISO loopback (`assign MISO = MOSI`).

```verilog
`timescale 1ns/1ps

module spi_tb;
    reg         clk, resetn;
    reg         isIO, mem_wstrb, mem_rstrb;
    reg  [29:0] mem_wordaddr;
    reg  [31:0] mem_wdata;
    wire [31:0] spi_rdata;
    wire        spi_sel;
    wire        SCLK, MOSI, CS_N;
    wire        MISO;

    assign MISO = MOSI;

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

    initial clk = 0;
    always  #5 clk = ~clk;

    task write_reg;
        input [29:0] addr;
        input [31:0] data;
        begin
            @(posedge clk); #1;
            isIO = 1; mem_wstrb = 1; mem_wordaddr = addr; mem_wdata = data;
            @(posedge clk); #1;
            mem_wstrb = 0; isIO = 0;
        end
    endtask

    task read_reg;
        input  [29:0] addr;
        output [31:0] val;
        begin
            @(posedge clk); #1;
            isIO = 1; mem_rstrb = 1; mem_wordaddr = addr;
            @(posedge clk); #1;
            val = spi_rdata; mem_rstrb = 0; isIO = 0;
        end
    endtask

    integer i;
    reg [31:0] rx_val, status_val;
    reg [7:0]  test_bytes [0:5];

    initial begin
        $dumpfile("spi_tb.vcd");
        $dumpvars(0, spi_tb);

        resetn = 0; isIO = 0; mem_wstrb = 0; mem_rstrb = 0;
        mem_wordaddr = 0; mem_wdata = 0;
        repeat (5) @(posedge clk);
        resetn = 1;
        repeat (2) @(posedge clk);

        test_bytes[0] = 8'hA5;
        test_bytes[1] = 8'h3C;
        test_bytes[2] = 8'hFF;
        test_bytes[3] = 8'h00;
        test_bytes[4] = 8'hAA;
        test_bytes[5] = 8'h55;

        $display("=== SPI Master Testbench ===");

        for (i = 0; i < 6; i = i + 1) begin
            $display("--- Transfer %0d: TX=0x%02X ---", i, test_bytes[i]);

            write_reg(30'd17, {24'h0, test_bytes[i]});
            write_reg(30'd16, {16'h0, 8'd4, 6'h0, 1'b1, 1'b1});

            begin : poll
                integer count;
                count = 0;
                forever begin
                    read_reg(30'd19, status_val);
                    count = count + 1;
                    if (status_val[1]) disable poll;
                    if (count > 100000) begin $display("TIMEOUT"); disable poll; end
                end
            end

            read_reg(30'd18, rx_val);
            write_reg(30'd19, 32'h2);

            if (rx_val[7:0] == test_bytes[i])
                $display("  PASS: RX=0x%02X", rx_val[7:0]);
            else
                $display("  FAIL: expected 0x%02X got 0x%02X", test_bytes[i], rx_val[7:0]);
        end

        $display("=== Testbench Complete ===");
        #100; $finish;
    end
endmodule
```

**Compile and run:**
```bash
iverilog -o spi_tb spi_tb.v spi_master.v
vvp spi_tb
gtkwave spi_tb.vcd &
```

**Expected output:**
```
=== SPI Master Testbench ===
--- Transfer 0: TX=0xA5 ---
  PASS: RX=0xA5
--- Transfer 1: TX=0x3C ---
  PASS: RX=0x3C
--- Transfer 2: TX=0xFF ---
  PASS: RX=0xFF
--- Transfer 3: TX=0x00 ---
  PASS: RX=0x00
--- Transfer 4: TX=0xAA ---
  PASS: RX=0xAA
--- Transfer 5: TX=0x55 ---
  PASS: RX=0x55
=== Testbench Complete ===
```

---

## 11. Quick Reference Cheat Sheet

### Minimum Transfer (5 operations)

```c
IO_OUT(SPI_TXDATA, tx);
IO_OUT(SPI_CTRL, (clkdiv << 8) | 0x3);
while (!(IO_IN(SPI_STATUS) & 0x2));
IO_OUT(SPI_STATUS, 0x2);
uint8_t rx = (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
```

### CLKDIV Quick Reference (12 MHz)

| Target SCLK | CLKDIV | Macro |
|-------------|--------|-------|
| 6 MHz | 0 | `SPI_CLKDIV_6MHZ` |
| 1.2 MHz | 4 | — |
| 1 MHz | 5 | `SPI_CLKDIV_1MHZ` |
| 500 kHz | 11 | `SPI_CLKDIV_500KHZ` |
| 250 kHz | 23 | `SPI_CLKDIV_250KHZ` |
| 100 kHz | 59 | `SPI_CLKDIV_100KHZ` |

### Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Writing CTRL before TXDATA | Old byte transmitted | Always write `SPI_TXDATA` first |
| Not clearing DONE | DONE appears immediately on next transfer | Write `0x2` to `SPI_STATUS` after each transfer |
| Wrong byte offset | Wrong register accessed | Offsets are 64/68/72/76, not word addresses 16/17/18/19 |
