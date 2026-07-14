# SPI Master IP — Integration Guide

**Target SoC:** FemtoRV32-style RISC-V (`riscv.v`) on VSDSquadron FM  
**Prerequisites:** Tasks 1–6 complete (GPIO and UART already integrated)

---

## Table of Contents

1. [Prerequisites and Environment](#1-prerequisites-and-environment)
2. [File Inventory](#2-file-inventory)
3. [Step 1 — Copy the IP File](#step-1--copy-the-ip-file)
4. [Step 2 — Update io.h](#step-2--update-ioh)
5. [Step 3 — Edit riscv.v (6 Changes)](#step-3--edit-riscvv-6-changes)
6. [Step 4 — Update PCF Pin Assignments](#step-4--update-pcf-pin-assignments)
7. [Step 5 — Write the Firmware](#step-5--write-the-firmware)
8. [Step 6 — Build Firmware Hex](#step-6--build-firmware-hex)
9. [Step 7 — Synthesize and Implement](#step-7--synthesize-and-implement)
10. [Step 8 — Flash to Hardware](#step-8--flash-to-hardware)
11. [Step 9 — Simulation Verification](#step-9--simulation-verification)
12. [Troubleshooting](#troubleshooting)
13. [Complete spi_master.v](#complete-spi_masterv)

---

## 1. Prerequisites and Environment

### Required Tools

| Tool | Purpose |
|------|---------|
| `iverilog` | Verilog simulation |
| `yosys` | Synthesis |
| `nextpnr-ice40` | Place and route |
| `icetime` | Timing analysis |
| `icepack` | Bitstream generation |
| `iceprog` | FPGA flash programmer |
| `riscv64-unknown-elf-gcc` | RISC-V firmware compiler |

### Verify Tools

```bash
iverilog -V
yosys --version
nextpnr-ice40 --version
riscv64-unknown-elf-gcc --version
```

### Working Directory

```
vsdfpga_labs/basicRISCV/
├── RTL/
│   ├── riscv.v           ← main SoC (to be modified)
│   ├── spi_master.v      ← NEW: copy here
│   └── VSDSquadronFM.pcf ← pin constraints (to be modified)
└── Firmware/
    ├── io.h              ← to be modified
    ├── spi_test.c        ← NEW: create this
    └── Makefile
```

---

## 2. File Inventory

| Action | File | Description |
|--------|------|-------------|
| **Create** | `RTL/spi_master.v` | SPI Master IP RTL |
| **Modify** | `Firmware/io.h` | Add SPI register macros |
| **Modify** | `RTL/riscv.v` | 6 edits to integrate the IP |
| **Modify** | `RTL/VSDSquadronFM.pcf` | Add SCLK/MOSI/MISO/CS_N pins |
| **Create** | `Firmware/spi_test.c` | Test firmware |

---

## Step 1 — Copy the IP File

```bash
cp spi_master.v vsdfpga_labs/basicRISCV/RTL/spi_master.v
ls vsdfpga_labs/basicRISCV/RTL/spi_master.v
```

---

## Step 2 — Update io.h

Open `Firmware/io.h` and add the four SPI macros after the existing GPIO definitions:

```c
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

#define IO_IN(port)       *(volatile uint32_t*)(IO_BASE + port)
#define IO_OUT(port,val)  *(volatile uint32_t*)(IO_BASE + port)=(val)
```

---

## Step 3 — Edit riscv.v (6 Changes)

### Change 1 — Include the SPI master file

Add after existing include statements:

```verilog
`include "spi_master.v"
```

### Change 2 — Add SPI ports to module SOC

```verilog
module SOC (
    input            RESET,
    output reg [4:0] LEDS,
    input            RXD,
    output           TXD,
    output           SCLK,
    output           MOSI,
    input            MISO,
    output           CS_N
);
```

### Change 3 — Instantiate SPI Master

Add after the GPIO block:

```verilog
wire [31:0] spi_rdata;
wire        spi_sel;

spi_master SPI(
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
```

### Change 4 — Guard the LEDS write path

Find:
```verilog
if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit] & !gpio_sel) begin
```

Replace with:
```verilog
if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit] & !gpio_sel & !spi_sel) begin
```

SPI word addresses 16–19 have bits overlapping with `IO_LEDS_bit`. Without this guard, writing SPI registers also triggers LED writes.

### Change 5 — Guard uart_valid

Find:
```verilog
wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit] & !gpio_sel;
```

Replace with:
```verilog
wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit] & !gpio_sel & !spi_sel;
```

Without this guard, writing `SPI_TXDATA` (word 17) spuriously transmits a UART character.

### Change 6 — Extend the IO_rdata read mux

Find:
```verilog
wire [31:0] IO_rdata =
    (mem_wordaddr[IO_UART_CNTL_bit] & !gpio_sel) ? {22'b0, !uart_ready, 9'b0}
    : gpio_sel ? gpio_rdata
    : 32'b0;
```

Replace with:
```verilog
wire [31:0] IO_rdata =
    (mem_wordaddr[IO_UART_CNTL_bit] & !gpio_sel & !spi_sel) ? {22'b0, !uart_ready, 9'b0}
    : gpio_sel  ? gpio_rdata
    : spi_sel   ? spi_rdata
    : 32'b0;
```

### Change 7 (Simulation only) — MISO loopback for BENCH mode

```verilog
`ifdef BENCH
    assign MISO = MOSI;
`endif
```

### Verify All Changes

```bash
grep -n "spi_master\|spi_sel\|spi_rdata\|SCLK\|MOSI\|MISO\|CS_N" RTL/riscv.v
```

---

## Step 4 — Update PCF Pin Assignments

Add four lines to `RTL/VSDSquadronFM.pcf`:

```
set_io  LEDS[0] 39
set_io  LEDS[1] 41
set_io  LEDS[2] 40
set_io  LEDS[3] 25
set_io  LEDS[4] 26
set_io  RESET 23
set_io  CLK 35
set_io  TXD 4
set_io  RXD 3
set_io  SCLK 6
set_io  MOSI 9
set_io  MISO 10
set_io  CS_N 11
```

### Pin Reference

| Signal | Pin | Notes |
|--------|-----|-------|
| `SCLK` | 6 | SPI clock output |
| `MOSI` | 9 | Master-Out Slave-In |
| `MISO` | 10 | Master-In Slave-Out |
| `CS_N` | 11 | Chip Select, active low |
| `CLK` | 35 | 12 MHz crystal input |

---

## Step 5 — Write the Firmware

Create `Firmware/spi_test.c`:

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

static uint8_t spi_transfer(uint8_t clkdiv, uint8_t tx) {
    IO_OUT(SPI_TXDATA, tx);
    IO_OUT(SPI_CTRL, ((uint32_t)clkdiv << 8) | 0x3);
    while (!(IO_IN(SPI_STATUS) & 0x2));
    IO_OUT(SPI_STATUS, 0x2);
    return (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
}

void main() {
    uint8_t rx;

    rx = spi_transfer(4, 0xA5);
    print_hex(rx);
    if (rx == 0xA5) print_str("PASS\n");

    rx = spi_transfer(4, 0x3C);
    print_hex(rx);
    if (rx == 0x3C) print_str("PASS\n");
}
```

---

## Step 6 — Build Firmware Hex

```bash
cd vsdfpga_labs/basicRISCV/Firmware
make spi_test.bram.hex
```

Expected:
```
Code size: 778 words ( total RAM size: 1536 words )
Occupancy: 50%
max_addr OK
SAVE HEX: spi_test.bram.hex
```

---

## Step 7 — Synthesize and Implement

```bash
cd vsdfpga_labs/basicRISCV/RTL
make
```

The Makefile runs:

```bash
yosys -p "synth_ice40 -top SOC -json SOC.json" riscv.v
nextpnr-ice40 --up5k --package sg48 --json SOC.json --pcf VSDSquadronFM.pcf --asc SOC.asc
icetime -p VSDSquadronFM.pcf -P sg48 -r SOC.timings -d up5k -t SOC.asc
icepack -s SOC.asc SOC.bin
```

Expected results:
```
ICESTORM_LC:   1136/5280   21%
ICESTORM_RAM:    16/30     53%
Max frequency: 19.54 MHz   PASS at 12 MHz
```

---

## Step 8 — Flash to Hardware

```bash
sudo bash -c 'echo "0403 6014" | tee /sys/bus/usb-serial/drivers/ftdi_sio/new_id'
sudo iceprog SOC.bin
```

Expected:
```
init..
cdone: high
reset..
cdone: low
flash ID: 0xEF 0x40 0x16 0x00
erase 64kB sector at 0x000000..
erase 64kB sector at 0x010000..
programming..
done.
reading..
VERIFY OK
cdone: high
Bye.
```

---

## Step 9 — Simulation Verification

### Standalone IP Testbench

```bash
cd vsdfpga_labs/basicRISCV/RTL
iverilog -o spi_tb spi_tb.v spi_master.v
vvp spi_tb
gtkwave spi_tb.vcd &
```

Expected:
```
TXDATA written: 0xA5
DONE set after 19 polls
RXDATA = 0xa5
PASS: SPI loopback correct
STATUS after clear = 0x00000000
```

### Full-SoC Simulation

```bash
iverilog -DBENCH -o soc_sim riscv.v
timeout 120 vvp soc_sim > /tmp/spi_out.txt 2>&1
cat /tmp/spi_out.txt | tr -cd '[:print:]\n' | grep -v "^t=" | cut -c1 | \
    awk 'NR%2==1' | tr -d '\n' | sed 's/PASS/\nPASS\n/g'
```

Expected:
```
000000A5
PASS
0000003C
PASS
```

---

## Troubleshooting

### Simulation Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `RXDATA = 0xxx` | Registers not reset in spi_master.v | Verify all registers initialized in `if (!resetn)` block |
| Simulation hangs forever | UART baud rate truncation | Use `BENCH baud = 200000`, not 1000000 |
| RXDATA = 0x00 in SoC sim | MISO floats as X | Add `` `ifdef BENCH assign MISO = MOSI; `endif `` in riscv.v |

### Synthesis Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `SB_HFOSC` unsupported | nextpnr version doesn't support internal oscillator | Use external CLK pin 35 |
| `UNKNOWN_FREQUENCY` | femtoPLL given input freq (12) | Use `CPU_FREQ=20` (PLL output frequency) |
| `SB_PLL40_PAD` error | PLL not connected to IO pad | Connect PLL input to CLK pin 35 directly |

### Hardware Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cdone: low` after flash | Invalid bitstream | Re-run `make`, check synthesis errors |
| RXDATA always 0x00 | MISO pin floating | Connect loopback wire MISO (pin 10) → MOSI (pin 9) for testing |
| Spurious UART output | Missing `& !spi_sel` on `uart_valid` | Apply Change 5 above |
| LEDs flashing on SPI writes | Missing `& !spi_sel` on LEDS path | Apply Change 4 above |

---

## Complete spi_master.v

```verilog
module spi_master (
    input  wire        clk,
    input  wire        resetn,
    input  wire        isIO,
    input  wire        mem_wstrb,
    input  wire        mem_rstrb,
    input  wire [29:0] mem_wordaddr,
    input  wire [31:0] mem_wdata,
    output reg  [31:0] spi_rdata,
    output wire        spi_sel,
    output reg         SCLK,
    output wire        MOSI,
    output reg         CS_N,
    input  wire        MISO
);
    wire sel_ctrl   = isIO & (mem_wordaddr[7:0] == 8'd16);
    wire sel_txdata = isIO & (mem_wordaddr[7:0] == 8'd17);
    wire sel_rxdata = isIO & (mem_wordaddr[7:0] == 8'd18);
    wire sel_status = isIO & (mem_wordaddr[7:0] == 8'd19);

    assign spi_sel = sel_ctrl | sel_txdata | sel_rxdata | sel_status;

    reg [15:0] clkdiv;
    reg        en;
    reg [7:0]  tx_data;
    reg [7:0]  rx_data;
    reg        busy;
    reg        done;
    reg [15:0] clk_cnt;
    reg [3:0]  bit_cnt;
    reg [7:0]  shift_reg;
    reg        sclk_en;

    assign MOSI = shift_reg[7];

    always @(posedge clk) begin
        if (!resetn) begin
            en        <= 0;
            clkdiv    <= 16'd4;
            tx_data   <= 8'h0;
            done      <= 0;
            busy      <= 0;
            sclk_en   <= 0;
            CS_N      <= 1;
            SCLK      <= 0;
            bit_cnt   <= 0;
            clk_cnt   <= 0;
            shift_reg <= 8'h0;
            rx_data   <= 8'h0;
        end else begin
            if (sel_ctrl & mem_wstrb) begin
                en     <= mem_wdata[0];
                clkdiv <= mem_wdata[15:8];
                if (mem_wdata[1] & !busy & mem_wdata[0]) begin
                    shift_reg <= tx_data;
                    bit_cnt   <= 0;
                    clk_cnt   <= 0;
                    busy      <= 1;
                    CS_N      <= 0;
                    SCLK      <= 0;
                    sclk_en   <= 1;
                end
            end
            if (sel_txdata & mem_wstrb) tx_data <= mem_wdata[7:0];
            if (sel_status & mem_wstrb & mem_wdata[1]) done <= 0;

            if (busy) begin
                if (clk_cnt < clkdiv) begin
                    clk_cnt <= clk_cnt + 1;
                end else begin
                    clk_cnt <= 0;
                    SCLK    <= ~SCLK;
                    if (!SCLK) begin
                        if (bit_cnt == 7) begin
                            rx_data   <= {shift_reg[6:0], MISO};
                            busy      <= 0;
                            done      <= 1;
                            CS_N      <= 1;
                            SCLK      <= 0;
                            sclk_en   <= 0;
                        end else begin
                            shift_reg <= {shift_reg[6:0], MISO};
                            bit_cnt   <= bit_cnt + 1;
                        end
                    end
                end
            end
        end
    end

    always @(*) begin
        if      (sel_ctrl)   spi_rdata = {16'h0, clkdiv, 6'h0, 1'b0, en};
        else if (sel_rxdata) spi_rdata = {24'h0, rx_data};
        else if (sel_status) spi_rdata = {30'h0, done, busy};
        else                 spi_rdata = 32'h0;
    end
endmodule
```
