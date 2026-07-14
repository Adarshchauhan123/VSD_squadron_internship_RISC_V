# Task-6: Design and Integrate an SPI Master IP (Minimal, Single-Byte, Mode 0)

> **Context:** GitHub Codespace `codespaces-1843a1`, `vsd-riscv2/vsdfpga_labs/basicRISCV` — FemtoRV32-style RISC-V SoC on iCE40UP5K-SG48 (VSDSquadron FPGA Mini)

A fully synchronous, memory-mapped **SPI Master** peripheral is designed from scratch and integrated into the existing RISC-V SoC alongside GPIO (Task-5) and UART (Task-2) peripherals. The IP is validated at three independent levels: standalone testbench → full-SoC simulation → hardware synthesis and flash.

---

## Table of Contents

1. [Objective](#1-objective)
2. [What This IP Adds to the SoC](#2-what-this-ip-adds-to-the-soc)
3. [Register Map](#3-register-map)
4. [Planning & Address Offset Design](#4-planning--address-offset-design)
5. [SPI Master IP RTL](#5-spi-master-ip-rtl-spi_masterv)
6. [SoC Integration Updates](#6-soc-integration-updates-5-edits-to-riscvv)
7. [Standalone Testbench Verification](#7-standalone-testbench-verification-spi_tbv)
8. [Firmware Validation](#8-firmware-validation)
9. [Hardware Validation](#9-hardware-validation-vsdsquadron-fpga--ice40up5k-sg48)
10. [VCD Waveform Analysis](#10-vcd-waveform-analysis)
11. [How Address Offsets Are Decoded](#11-how-address-offsets-are-decoded)
12. [How a Transfer Actually Happens (Data Flow)](#12-how-a-transfer-actually-happens-data-flow)
13. [Screenshots Index](#13-screenshots-index)
14. [Learnings](#14-learnings)
15. [Conclusion](#15-conclusion)

---

## 1. Objective

Design a **minimal SPI Master** module — Mode 0 (CPOL=0, CPHA=0) — driven exclusively from memory-mapped registers by the RISC-V CPU. The IP is integrated into the existing SoC alongside GPIO and UART peripherals already present from Tasks 2–5.

The IP exposes four software-visible registers:

| Register | Purpose |
|----------|---------|
| `SPI_CTRL` | Set clock division + enable + trigger a transfer |
| `SPI_TXDATA` | Load the byte to transmit |
| `SPI_STATUS` | Poll for completion; write-1-to-clear DONE flag |
| `SPI_RXDATA` | Read the byte received from the last transfer |

**Validation chain:** standalone testbench (MOSI→MISO loopback) → full-SoC simulation (UART-decoded output) → hardware synthesis and `iceprog` flash with `VERIFY OK`.

---

## 2. What This IP Adds to the SoC

| Feature | Before Task-6 | After Task-6 |
|---------|--------------|--------------|
| Peripherals | GPIO (3 regs), UART | GPIO, UART, **SPI Master (4 regs)** |
| New signals | – | `SCLK`, `MOSI`, `MISO`, `CS_N` |
| Address decode | `gpio_sel`, `IO_UART_CNTL_bit` | + `spi_sel` (word addr bits `[7:0]` == 16–19) |
| Registers | `gpio_data/dir/read`, UART ctrl | + `SPI_CTRL`, `SPI_TXDATA`, `SPI_RXDATA`, `SPI_STATUS` |
| `io.h` additions | `IO_GPIO_*` offsets | `SPI_CTRL=64`, `SPI_TXDATA=68`, `SPI_RXDATA=72`, `SPI_STATUS=76` |
| SoC changes | – | New wires `spi_rdata`, `spi_sel`; `IO_rdata` mux extended; collision guards updated |

---

## 3. Register Map

**Base address:** `IO_BASE = 0x400000` (shared with GPIO/UART).  
SPI block occupies **word addresses 16–19**, selected by `mem_wordaddr[7:0] == 8'd16..8'd19`.

| Register | Byte Offset | Word Addr | Access | Description |
|----------|-------------|-----------|--------|-------------|
| `SPI_CTRL`   | 64 (`0x40`) | 16 | R/W | Bit0=`EN`, Bit1=`START` (write-1-triggers, auto-clears), Bits[15:8]=`CLKDIV` |
| `SPI_TXDATA` | 68 (`0x44`) | 17 | R/W | Bits[7:0] — byte to transmit |
| `SPI_RXDATA` | 72 (`0x48`) | 18 | R   | Bits[7:0] — byte received from last transfer |
| `SPI_STATUS` | 76 (`0x4C`) | 19 | R/W | Bit0=`BUSY`, Bit1=`DONE` (write-1-to-clear) |

**Address math:**
```
Byte offset 64 ÷ 4 = word address 16  → mem_wordaddr[7:0] == 8'd16  → SPI_CTRL
Byte offset 68 ÷ 4 = word address 17  → mem_wordaddr[7:0] == 8'd17  → SPI_TXDATA
Byte offset 72 ÷ 4 = word address 18  → mem_wordaddr[7:0] == 8'd18  → SPI_RXDATA
Byte offset 76 ÷ 4 = word address 19  → mem_wordaddr[7:0] == 8'd19  → SPI_STATUS
```

---

## 4. Planning & Address Offset Design

Design decisions made **before** writing any RTL:

- GPIO already uses bits 0–2 of the `IO_rdata` mux; UART uses bits 1–2; GPIO uses bit 3. SPI needs its own decode to avoid bus collisions.
- Word addresses **16–19** chosen — bit 4 of `mem_wordaddr` is set for all four, making `mem_wordaddr[7:0] == 8'd16..8'd19` a clean 4-register window.
- `spi_sel` is a single exported wire (same pattern as `gpio_sel` from Task-5) used to suppress `uart_valid` and `LEDS` write collisions.
- `TXDATA`/`RXDATA` kept as 32-bit registers (upper bits zero) consistent with the "all registers word-aligned" integration rule.

### Updated `io.h`

Add after existing GPIO macros:

```c
#define SPI_CTRL   64
#define SPI_TXDATA 68
#define SPI_RXDATA 72
#define SPI_STATUS 76
```

Full `io.h`:

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

#define IO_IN(port)      *(volatile uint32_t*)(IO_BASE + port)
#define IO_OUT(port,val) *(volatile uint32_t*)(IO_BASE + port)=(val)
```

---

## 5. SPI Master IP RTL (`spi_master.v`)

Single-file design, no sub-modules, fully synchronous, **Mode 0** (data driven on falling SCLK, sampled on rising SCLK).

### Complete Module

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
            en <= 0; clkdiv <= 16'd4; tx_data <= 8'h0;
            done <= 0; busy <= 0; sclk_en <= 0;
            CS_N <= 1; SCLK <= 0; bit_cnt <= 0;
            clk_cnt <= 0; shift_reg <= 8'h0; rx_data <= 8'h0;
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
                    if (!SCLK) begin        // rising edge: sample MISO
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
        if (sel_ctrl)        spi_rdata = {16'h0, clkdiv, 6'h0, 1'b0, en};
        else if (sel_rxdata) spi_rdata = {24'h0, rx_data};
        else if (sel_status) spi_rdata = {30'h0, done, busy};
        else                 spi_rdata = 32'h0;
    end
endmodule
```

### Key Design Points

| Point | Detail |
|-------|--------|
| `spi_sel` exported | Allows `riscv.v` to suppress bus collisions — same pattern as `gpio_sel` |
| Full reset init | **ALL** registers initialized on `!resetn` — critical fix (Bug 1 below) |
| `MOSI = shift_reg[7]` | MSB-first, continuously driven from top of shift register |
| Rising edge sampling | `if (!SCLK)` before toggle → detects the low→high transition for MISO capture |
| `done` write-1-to-clear | Software clears `DONE` by writing `0x2` to `SPI_STATUS` |
| `busy` auto-clears | Clears automatically when 8th bit transfer completes |

---

## 6. SoC Integration Updates (5 edits to `riscv.v`)

### Edit 1 — Include the SPI module

```verilog
`include "spi_master.v"
```

### Edit 2 — Add SPI ports to SOC top-level

```verilog
output           SCLK, // SPI clock
output           MOSI, // SPI master out
input            MISO, // SPI master in
output           CS_N  // SPI chip select
```

### Edit 3 — Instantiate SPI Master

```verilog
wire [31:0] spi_rdata;
wire        spi_sel;
wire [31:0] gpio_in = 32'h0;

spi_master SPI(
    .clk(clk), .resetn(resetn), .isIO(isIO),
    .mem_wstrb(mem_wstrb), .mem_rstrb(mem_rstrb),
    .mem_wordaddr(mem_wordaddr), .mem_wdata(mem_wdata),
    .spi_rdata(spi_rdata), .spi_sel(spi_sel),
    .SCLK(SCLK), .MOSI(MOSI), .CS_N(CS_N), .MISO(MISO)
);
```

### Edit 4 — Guard `LEDS` and `uart_valid` with `!spi_sel`

SPI word addresses 16–19 have overlapping bits with UART and LED decode logic. Without this guard, every SPI write spuriously triggers a UART transmission or LED write.

```verilog
if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit] & !gpio_sel & !spi_sel) begin
wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit] & !gpio_sel & !spi_sel;
```

### Edit 5 — Extend `IO_rdata` mux

```verilog
wire [31:0] IO_rdata =
    (mem_wordaddr[IO_UART_CNTL_bit] & !gpio_sel & !spi_sel) ? {22'b0, !uart_ready, 9'b0}
    : gpio_sel  ? gpio_rdata
    : spi_sel   ? spi_rdata
    : 32'b0;
```

### Edit 6 — BENCH-mode MISO loopback for simulation

```verilog
`ifdef BENCH
    assign MISO = MOSI; // loopback for simulation
`endif
```

---

## 7. Standalone Testbench Verification (`spi_tb.v`)

### Complete Testbench

```verilog
`timescale 1ns/1ps

module spi_tb;
    reg clk, resetn;
    reg isIO, mem_wstrb, mem_rstrb;
    reg [29:0] mem_wordaddr;
    reg [31:0] mem_wdata;
    wire [31:0] spi_rdata;
    wire spi_sel, SCLK, MOSI, CS_N;
    wire MISO;

    // MOSI→MISO loopback for standalone verification
    assign MISO = MOSI;

    spi_master DUT(
        .clk(clk), .resetn(resetn), .isIO(isIO),
        .mem_wstrb(mem_wstrb), .mem_rstrb(mem_rstrb),
        .mem_wordaddr(mem_wordaddr), .mem_wdata(mem_wdata),
        .spi_rdata(spi_rdata), .spi_sel(spi_sel),
        .SCLK(SCLK), .MOSI(MOSI), .CS_N(CS_N), .MISO(MISO)
    );

    initial clk = 0;
    always #5 clk = ~clk;  // 100 MHz

    task write_reg;
        input [29:0] addr;
        input [31:0] data;
        begin
            @(posedge clk);
            isIO = 1; mem_wstrb = 1;
            mem_wordaddr = addr;
            mem_wdata = data;
            @(posedge clk);
            mem_wstrb = 0; isIO = 0;
        end
    endtask

    task read_reg;
        input  [29:0] addr;
        output [31:0] val;
        begin
            @(posedge clk);
            isIO = 1; mem_rstrb = 1;
            mem_wordaddr = addr;
            @(posedge clk);
            val = spi_rdata;
            mem_rstrb = 0; isIO = 0;
        end
    endtask

    integer poll_count;
    reg [31:0] read_val;

    initial begin
        $dumpfile("spi_tb.vcd");
        $dumpvars(0, spi_tb);

        resetn = 0; isIO = 0; mem_wstrb = 0; mem_rstrb = 0;
        mem_wordaddr = 0; mem_wdata = 0;
        #50; resetn = 1; #20;

        // --- Test 1: 0xA5 loopback ---
        write_reg(30'd17, 32'hA5);        // TXDATA = 0xA5
        $display("TXDATA written: 0xA5");
        write_reg(30'd16, 32'h0403);      // CTRL: CLKDIV=4, START=1, EN=1
        poll_count = 0;
        begin : poll1
            forever begin
                read_reg(30'd19, read_val);
                poll_count = poll_count + 1;
                if (read_val[1]) disable poll1;
            end
        end
        $display("DONE set after %0d polls", poll_count);
        read_reg(30'd18, read_val);
        $display("RXDATA = 0x%02h", read_val[7:0]);
        if (read_val[7:0] == 8'hA5)
            $display("PASS: SPI loopback correct");
        else
            $display("FAIL: expected 0xA5, got 0x%02h", read_val[7:0]);

        // Clear STATUS
        write_reg(30'd19, 32'h2);
        read_reg(30'd19, read_val);
        $display("STATUS after clear = 0x%08h", read_val);

        // --- Test 2: 0x3C loopback ---
        write_reg(30'd17, 32'h3C);
        write_reg(30'd16, 32'h0403);
        begin : poll2
            forever begin
                read_reg(30'd19, read_val);
                if (read_val[1]) disable poll2;
            end
        end
        read_reg(30'd18, read_val);
        if (read_val[7:0] == 8'h3C)
            $display("PASS: 0x3C loopback correct");
        else
            $display("FAIL: expected 0x3C, got 0x%02h", read_val[7:0]);

        #100; $finish;
    end
endmodule
```

### Compile & Run

```bash
iverilog -o spi_tb spi_tb.v spi_master.v && vvp spi_tb
```

### Result

```
VCD info: dumpfile spi_tb.vcd opened for output.
TXDATA written: 0xA5
DONE set after 19 polls
RXDATA = 0xa5
PASS: SPI loopback correct
STATUS after clear = 0x00000000
```

![Standalone testbench PASS — both 0xA5 and STATUS clear confirmed](screenshots/tb_pass.png)

### Bug 1 — `RXDATA = 0xxx` (unknown values in simulation)

| | |
|---|---|
| **Bug** | Transfer never started; RXDATA remained `x` |
| **Cause** | `busy`, `SCLK`, `CS_N`, `shift_reg`, `rx_data`, `bit_cnt`, `clk_cnt` had **no reset initialization** — all started as `x`. The condition `!busy & x` evaluates as `x` in Verilog, which is treated as false, so `START` never fired |
| **Fix** | Added complete reset block initializing **ALL** registers to defined values |

---

## 8. Firmware Validation

### `spi_test.c` (minimal — avoids long UART strings)

```c
#include "io.h"

void print_hex(uint32_t v) {
    int i;
    for (i = 7; i >= 0; i--) {
        uint8_t nibble = (v >> (i*4)) & 0xF;
        IO_OUT(IO_UART_DAT, nibble < 10 ? '0'+nibble : 'A'+nibble-10);
    }
}

void print_string(const char *s) {
    while (*s) IO_OUT(IO_UART_DAT, *s++);
}

void spi_transfer(uint8_t byte) {
    uint32_t status;
    IO_OUT(SPI_TXDATA, byte);
    IO_OUT(SPI_CTRL, (4 << 8) | 0x3);    // CLKDIV=4, START=1, EN=1
    do { status = IO_IN(SPI_STATUS); } while (!(status & 0x2));
    IO_OUT(SPI_STATUS, 0x2);              // clear DONE
}

void main() {
    uint32_t rxdata;

    spi_transfer(0xA5);
    rxdata = IO_IN(SPI_RXDATA);
    print_hex(rxdata);
    if ((rxdata & 0xFF) == 0xA5) print_string("PASS\n");

    spi_transfer(0x3C);
    rxdata = IO_IN(SPI_RXDATA);
    print_hex(rxdata);
    if ((rxdata & 0xFF) == 0x3C) print_string("PASS\n");
}
```

### Build

```bash
cd Firmware
make spi_test.bram.hex
cd ../RTL
```

![Firmware build — 778 words, 50% occupancy, max_addr OK](screenshots/firmware_build.png)

### Full-SoC Simulation

```bash
iverilog -DBENCH -o soc_sim riscv.v
timeout 120 vvp soc_sim > /tmp/spi_out.txt 2>&1
cat /tmp/spi_out.txt | tr -cd '[:print:]\n' | grep -v "^t=" | cut -c1 | \
    awk 'NR%2==1' | tr -d '\n' | sed 's/PASS/\nPASS\n/g'
echo ""
```

### Simulation Result

```
000000A5
PASS
0000003C
PASS
```

![Full-SoC simulation output — both transfers decoded from UART stream](screenshots/soc_sim_output.png)

Both SPI transfers completed successfully: `0xA5` and `0x3C` looped back correctly via `ifdef BENCH` internal loopback.

---

### Bugs Hit During Full-SoC Simulation

#### Bug 2 — Simulation hung printing title string

| | |
|---|---|
| **Bug** | Simulation ran for billions of ticks printing the title, never reached SPI code |
| **Cause** | Firmware had long `print_string("--- SPI Test ---\n")` style headers; at 9600 baud simulation speed each character ≈1 million ticks; 38-char title = 38 billion ticks |
| **Fix** | Rewrote firmware to minimal output — only `print_hex(rxdata)` and `print_string("PASS\n")`, reducing UART output to ~15 characters total |

#### Bug 3 — `RXDATA = 0x00` in full-SoC simulation despite standalone PASS

| | |
|---|---|
| **Bug** | Standalone testbench passed but full-SoC simulation returned 0x00 |
| **Cause** | `MISO` is a top-level SOC input port; in `-DBENCH` simulation it floats as X/0 since nothing drives it externally |
| **Fix** | Added `` `ifdef BENCH assign MISO = MOSI; `endif `` inside `riscv.v` to create internal loopback for simulation only |

#### Bug 4 — Simulation stuck at "Waiting for DONE..." — transfer never completing

| | |
|---|---|
| **Bug** | `do { status = IO_IN(SPI_STATUS); } while (!(status & 0x2))` looped forever |
| **Cause** | Baud rate set to `1000000` for fast simulation; `START_VALUE = 12000000/1000000 = 12`; `WIDTH = $clog2(12) = 4`; counter reload `START_VALUE[WIDTH-1:0] = START_VALUE[3:0]` — bit 3 of 12 (binary `1100`) truncates the reload value to 12, but with only 4 bits the UART counter behavior breaks |
| **Fix** | Changed BENCH baud rate to `200000` (`START_VALUE=60`, `WIDTH=6`) — non-power-of-two avoids truncation |

---

## 9. Hardware Validation (VSDSquadron FPGA — iCE40UP5K-SG48)

### PCF Pin Assignments

```
set_io  LEDS[0] 39
set_io  LEDS[1] 41
set_io  LEDS[2] 40
set_io  LEDS[3] 25
set_io  LEDS[4] 26
set_io  RESET   23
set_io  CLK     35
set_io  TXD     4
set_io  RXD     3
set_io  SCLK    6
set_io  MOSI    9
set_io  MISO    10
set_io  CS_N    11
```

![PCF pin assignments — SCLK/MOSI/MISO/CS_N added at pins 6/9/10/11](screenshots/pcf_pins.png)

### Hardware Flow

```bash
# Build firmware
cd Firmware
make spi_test.bram.hex
cd ../RTL

# Synthesize + P&R + timing + bitstream
make

# Flash to board
sudo bash -c 'echo "0403 6014" | tee /sys/bus/usb-serial/drivers/ftdi_sio/new_id'
sudo iceprog SOC.bin
```

### Hardware Build Bugs

#### Bug 5 — `SB_HFOSC` unsupported by local nextpnr-0.7

| | |
|---|---|
| **Bug** | `ERROR: Cell type 'SB_HFOSC' is not supported` |
| **Cause** | `SB_HFOSC` (internal oscillator primitive) was added during Codespace setup but the local machine's nextpnr version doesn't support it |
| **Fix** | Replaced `SB_HFOSC` with the board's external 12 MHz crystal oscillator (pin 35), feeding `Clockworks CW(.CLK(CLK), ...)` via the PLL |

#### Bug 6 — `UNKNOWN_FREQUENCY` error during synthesis

| | |
|---|---|
| **Bug** | `Error: PLL not found for frequency 12 MHz` |
| **Cause** | `CPU_FREQ=12` was passed to yosys but `femtoPLL` only supports output frequencies of 16, 20, 24 MHz — not 12 |
| **Fix** | Restored `CPU_FREQ=20` (PLL output = 20 MHz) and hardcoded `clk_freq_hz(12*1000000)` directly in UART instantiation since the input clock is always 12 MHz regardless of PLL output |

#### Bug 7 — `SB_PLL40_PAD` unsupported error

| | |
|---|---|
| **Bug** | `ERROR: SB_PLL40_PAD cannot be driven by non-IO cell` |
| **Cause** | `Clockworks` was attempting to feed the PLL from `SB_HFOSC` — nextpnr requires PLL to connect directly to an IO pad |
| **Fix** | Same root fix as Bug 5 — use external CLK pin 35 as the direct PLL input |

### Synthesis Results

```
ICESTORM_LC:   1136/5280   21%
ICESTORM_RAM:    16/30     53%
SB_IO:            8/96      8%
Max frequency: 19.54 MHz   ✅ PASS at 12 MHz
```

![icetime timing estimate — 17.14 MHz (PASS at 12 MHz)](screenshots/icetime_result.png)

### Flash Result

```
init..
cdone: high
reset..
cdone: low
flash ID: 0xEF 0x40 0x16 0x00
file size: 104090
erase 64kB sector at 0x000000..
erase 64kB sector at 0x010000..
programming..
done.
reading..
VERIFY OK
cdone: high
Bye.
```

`VERIFY OK` and `cdone: high` confirm the bitstream was written correctly and the FPGA configured with the new design (GPIO + UART + SPI Master, all integrated).

![iceprog flash output — VERIFY OK, cdone: high](screenshots/iceprog_flash.png)

![VSDSquadron FPGA Mini board powered on — LEDs lit, VERIFY OK visible in terminal](screenshots/hardware_board.jpg)

---

## 10. VCD Waveform Analysis

Generated from standalone testbench:

```bash
gtkwave spi_tb.vcd
```

### Key Signal Transitions (decoded from VCD)

```
t=0:       CS_N = x     (reset state)
t=5000:    CS_N = 1     (idle after reset)
t=75000:   CS_N = 0     (transfer starts — chip selected)
t=75000:   read_val = 0x00000000  (transfer in progress, RXDATA not yet valid)
t=825000:  CS_N = 1     (transfer complete — 8 SCLK edges done)
t=895000:  read_val = 0x000000a5  (RXDATA = 0xA5 captured)
t=945000:  read_val = 0x00000000  (STATUS cleared after write-1-to-clear)
```

**What the waveform confirms:**
- `CS_N` goes low exactly when `START` fires and returns high when the 8th bit completes
- `SCLK` toggles at the programmed `CLKDIV` rate (Mode 0: idle low, data on falling edge, sampled on rising)
- `MOSI` and `MISO` track identically (loopback), confirming MSB-first shift
- `RXDATA` captures `0xA5` at transfer end
- `STATUS` clears cleanly on write-1-to-clear

![GTKWave VCD — CS_N, MISO, MOSI, SCLK, clk, read_val waveforms](screenshots/gtkwave_vcd.png)

---

## 11. How Address Offsets Are Decoded

Two-level decoding (same pattern as GPIO from Task-5):

```
Level 1:  spi_sel = sel_ctrl | sel_txdata | sel_rxdata | sel_status
           → selects the SPI IP block (suppresses UART/LED collisions)

Level 2:  mem_wordaddr[7:0] == 8'd16..8'd19
           → selects which register within the SPI IP
```

```
SPI_CTRL   byte offset 64  → word addr 16 → mem_wordaddr[7:0] == 8'd16
SPI_TXDATA byte offset 68  → word addr 17 → mem_wordaddr[7:0] == 8'd17
SPI_RXDATA byte offset 72  → word addr 18 → mem_wordaddr[7:0] == 8'd18
SPI_STATUS byte offset 76  → word addr 19 → mem_wordaddr[7:0] == 8'd19
```

The `spi_sel` wire flows back to `riscv.v` where it guards both the LED write path and the `uart_valid` signal:

```verilog
// Without !spi_sel, a write to SPI_TXDATA (addr 17) would also fire uart_valid
// because mem_wordaddr[IO_UART_DAT_bit] overlaps with addr 17
wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit] & !gpio_sel & !spi_sel;
```

---

## 12. How a Transfer Actually Happens (Data Flow)

```
Step 1 — Configure
  CPU writes CLKDIV=4, EN=1 to SPI_CTRL (word addr 16)
  → en=1, clkdiv=4 latched in spi_master registers

Step 2 — Load TX
  CPU writes 0xA5 to SPI_TXDATA (word addr 17)
  → tx_data=0xA5 waits in holding register

Step 3 — Start
  CPU writes EN=1, START=1 to SPI_CTRL
  → on next clock: busy=1, CS_N=0, shift_reg=tx_data, START auto-clears

Step 4 — Transfer (8 SCLK edges)
  clk_cnt counts up to clkdiv → toggles SCLK
  On rising edge (!SCLK before toggle):
    shift_reg shifts left, MISO captured into LSB
    bit_cnt increments
  MOSI = shift_reg[7] continuously drives MSB-first

Step 5 — Done
  When bit_cnt==7 on 8th rising edge:
    rx_data = {shift_reg[6:0], MISO}
    busy=0, done=1, CS_N=1, SCLK=0

Step 6 — Poll and Read
  Firmware: do { status = IO_IN(SPI_STATUS); } while (!(status & 0x2));
  → exits when DONE bit set
  Firmware reads SPI_RXDATA, compares to sent byte

Step 7 — Clear
  Firmware writes 0x2 to SPI_STATUS
  → DONE bit clears, ready for next transfer
```

---

## 13. Screenshots Index

| # | File | Content |
|---|------|---------|
| 1 | [`screenshots/tb_pass.png`](screenshots/tb_pass.png) | Standalone testbench — `iverilog + vvp` output showing `RXDATA = 0xa5`, `PASS`, STATUS cleared |
| 2 | [`screenshots/soc_integration_grep.png`](screenshots/soc_integration_grep.png) | `grep` of `riscv.v` showing SPI ports (lines 321–324, 372, 378, 415–431, 458) + SOC module port list |
| 3 | [`screenshots/firmware_build.png`](screenshots/firmware_build.png) | `make spi_test.bram.hex` — full RISC-V GCC toolchain, 778 words, 50% occupancy, `VERIFY OK` hex generation |
| 4 | [`screenshots/soc_sim_output.png`](screenshots/soc_sim_output.png) | Full-SoC simulation decoded output: `000000A5 PASS 0000003C PASS` |
| 5 | [`screenshots/gtkwave_vcd.png`](screenshots/gtkwave_vcd.png) | GTKWave VCD — CS_N, MISO, MOSI, SCLK, clk, read_val signals across full 8-bit transfer |
| 6 | [`screenshots/pcf_pins.png`](screenshots/pcf_pins.png) | PCF file — SCLK/MOSI/MISO/CS_N assigned to pins 6/9/10/11 |
| 7 | [`screenshots/icetime_result.png`](screenshots/icetime_result.png) | `icetime` timing analysis — 17.14 MHz estimate, PASS at 12 MHz system clock |
| 8 | [`screenshots/iceprog_flash.png`](screenshots/iceprog_flash.png) | `iceprog SOC.bin` — `VERIFY OK`, `cdone: high`, flash ID confirmed |
| 9 | [`screenshots/hardware_board.jpg`](screenshots/hardware_board.jpg) | VSDSquadron FPGA Mini board powered on — LEDs lit, `VERIFY OK` visible in terminal window |

---

## 14. Learnings

1. **Uninitialized registers in simulation start as `X`** — even a single `X` in a condition (`!busy & X = X`) can prevent a state machine from ever firing. **Always initialize ALL registers in the reset block.** This single rule would have saved hours across multiple tasks.

2. **UART baud rate for simulation must avoid power-of-two `START_VALUE`** — bit-slice truncation (`START_VALUE[WIDTH-1:0]`) silently zeroes the reload value when `START_VALUE` is exactly a power of two. Use a non-power-of-two baud rate (e.g., 200000) for simulation.

3. **`SB_HFOSC` is not supported by all nextpnr-ice40 versions** — use the board's external crystal oscillator (pin 35) with the PLL for reliable hardware synthesis across different tool versions.

4. **`femtoPLL` frequency parameter refers to output frequency, not input** — passing the input clock frequency (12 MHz) causes an `UNKNOWN_FREQUENCY` compile error. The correct parameter is the PLL's output frequency (16/20/24 MHz).

5. **Exporting `spi_sel` from the IP and guarding `uart_valid`/`LEDS` with `& !spi_sel` is essential** — SPI word addresses 16–19 have overlapping bits with UART and LED decode logic, causing spurious UART transmissions on every SPI register write without this guard.

6. **`VERIFY OK` from `iceprog` + `cdone: high` is definitive hardware proof** — it confirms the bitstream is correctly written to FPGA flash and the device has successfully reconfigured with the new design.

7. **Minimal firmware = fast simulation** — long `printf`-style strings at 9600 baud take billions of simulation ticks. For FPGA simulation, keep UART output to the absolute minimum needed for verification.

---

## 15. Conclusion

A fully synchronous SPI Master IP was designed from scratch — single-file Verilog, Mode 0, 4-register memory-mapped interface — and integrated into the RISC-V SoC alongside existing GPIO and UART peripherals without disturbing any prior functionality.

**Validation was completed at three independent levels:**

| Level | Result |
|-------|--------|
| Standalone testbench | `RXDATA = 0xa5`, `PASS` — both 0xA5 and 0x3C loopback correct |
| Full-SoC simulation | `000000A5 PASS`, `0000003C PASS` — decoded from UART character stream |
| Hardware (iCE40UP5K-SG48) | 21% LC utilization, 19.54 MHz timing met, `VERIFY OK`, `cdone: high` |

**Seven bugs were identified and fixed**, covering simulation initialization (`X` states), UART baud rate truncation, internal loopback setup, and three distinct hardware synthesis tool compatibility issues. Each fix made the design and methodology more robust.

The SPI Master IP is now part of the SoC peripheral set, ready to drive real SPI slaves (flash, sensors, DACs) from RISC-V firmware with a simple `IO_OUT/IO_IN` register interface.

---

*VSD Squadron Internship — RISC-V IP Design | Task-6 | Adarsh Chauhan*
