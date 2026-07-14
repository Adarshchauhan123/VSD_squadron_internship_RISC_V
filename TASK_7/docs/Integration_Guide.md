# SPI Master IP — Integration Guide

**IP Name:** SPI Master  
**Version:** 1.0  
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

| Tool | Version | Purpose |
|------|---------|---------|
| `iverilog` | 11.x or later | Verilog simulation |
| `vvp` | bundled with iverilog | VCD simulation runner |
| `yosys` | 0.9+ | Synthesis |
| `nextpnr-ice40` | 0.4+ | Place and route |
| `icetime` | bundled with icestorm | Timing analysis |
| `icepack` | bundled with icestorm | Bitstream generation |
| `iceprog` | bundled with icestorm | FPGA flash programmer |
| `riscv64-unknown-elf-gcc` | any recent | RISC-V firmware compiler |

### Verify Environment

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
│   ├── riscv.v          ← main SoC file (to be modified)
│   ├── spi_master.v     ← NEW: copy here
│   └── VSDSquadronFM.pcf ← pin constraints (to be modified)
└── Firmware/
    ├── io.h             ← to be modified
    ├── spi_test.c       ← NEW: create this
    └── Makefile
```

---

## 2. File Inventory

Files you will create or modify in this integration:

| Action | File | Description |
|--------|------|-------------|
| **Create** | `RTL/spi_master.v` | SPI Master IP RTL |
| **Modify** | `Firmware/io.h` | Add SPI register macros |
| **Modify** | `RTL/riscv.v` | 6 edits to integrate the IP |
| **Modify** | `RTL/VSDSquadronFM.pcf` | Add SCLK/MOSI/MISO/CS_N pins |
| **Create** | `Firmware/spi_test.c` | Test firmware |

---

## Step 1 — Copy the IP File

Copy `spi_master.v` to the RTL directory:

```bash
cp spi_master.v vsdfpga_labs/basicRISCV/RTL/spi_master.v
```

Verify:
```bash
ls vsdfpga_labs/basicRISCV/RTL/spi_master.v
```

---

## Step 2 — Update io.h

Open `Firmware/io.h` and add the four SPI macros **after** the existing GPIO definitions:

### Before

```c
#include <stdint.h>

#define IO_BASE       0x400000
#define IO_LEDS       4
#define IO_UART_DAT   8
#define IO_UART_CNTL  16
#define IO_GPIO_DATA  32
#define IO_GPIO_DIR   36
#define IO_GPIO_READ  40

#define IO_IN(port)       *(volatile uint32_t*)(IO_BASE + port)
#define IO_OUT(port,val)  *(volatile uint32_t*)(IO_BASE + port)=(val)
```

### After (add the 4 highlighted lines)

```c
#include <stdint.h>

#define IO_BASE       0x400000
#define IO_LEDS       4
#define IO_UART_DAT   8
#define IO_UART_CNTL  16
#define IO_GPIO_DATA  32
#define IO_GPIO_DIR   36
#define IO_GPIO_READ  40
#define SPI_CTRL      64     /* word addr 16 — Control: EN, START, CLKDIV */
#define SPI_TXDATA    68     /* word addr 17 — Transmit byte              */
#define SPI_RXDATA    72     /* word addr 18 — Receive byte (read-only)   */
#define SPI_STATUS    76     /* word addr 19 — BUSY (r), DONE (r/w1c)     */

#define IO_IN(port)       *(volatile uint32_t*)(IO_BASE + port)
#define IO_OUT(port,val)  *(volatile uint32_t*)(IO_BASE + port)=(val)
```

---

## Step 3 — Edit riscv.v (6 Changes)

Open `RTL/riscv.v`. Make the following 6 changes **in order**. Each change is shown with the exact lines to find and what to add.

---

### Change 1 — Include the SPI master file

**Find** the line where other modules are included (near the top of `riscv.v`, look for the last `` `include `` statement). Add the new include immediately after it:

```verilog
`include "spi_master.v"
```

**Exact placement** — after existing includes, before `module SOC`:
```verilog
`include "clockworks.v"
`include "emitter_uart.v"
`include "gpio_ip.v"
`include "spi_master.v"     // ← ADD THIS LINE
```

---

### Change 2 — Add SPI ports to module SOC

**Find** the `module SOC (` port list. Locate the existing output/input signals and add the four SPI signals. Find the closing `);` of the port list.

**Before:**
```verilog
module SOC (
    //  input        CLK,
    input            RESET,
    output reg [4:0] LEDS,
    input            RXD,
    output           TXD
);
```

**After:**
```verilog
module SOC (
    //  input        CLK,
    input            RESET,
    output reg [4:0] LEDS,
    input            RXD,
    output           TXD,
    output           SCLK,  // SPI clock
    output           MOSI,  // SPI master out, slave in
    input            MISO,  // SPI master in, slave out
    output           CS_N   // SPI chip select (active low)
);
```

---

### Change 3 — Declare wires and instantiate SPI Master

**Find** the section where GPIO is instantiated (search for `gpio_ip GPIO`). Add the SPI wire declarations and instantiation immediately after the GPIO block:

```verilog
// ─── SPI Master ────────────────────────────────────────────────────────────
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

---

### Change 4 — Guard the LEDS write path

**Find** the LEDS register write. It will look like:

```verilog
if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit] & !gpio_sel) begin
```

**Add `& !spi_sel`:**

```verilog
if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit] & !gpio_sel & !spi_sel) begin
```

**Why:** SPI word addresses 16–19 have bits that overlap with `IO_LEDS_bit` decode. Without this guard, a write to `SPI_TXDATA` (word 17) also triggers the LED register.

---

### Change 5 — Guard uart_valid

**Find** the `uart_valid` wire declaration:

```verilog
wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit] & !gpio_sel;
```

**Add `& !spi_sel`:**

```verilog
wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit] & !gpio_sel & !spi_sel;
```

**Why:** `IO_UART_DAT_bit` overlaps with SPI register word addresses. Without this guard, a write to SPI registers would spuriously transmit a UART character.

---

### Change 6 — Extend the IO_rdata read mux

**Find** the `IO_rdata` mux. It will look like:

```verilog
wire [31:0] IO_rdata =
    (mem_wordaddr[IO_UART_CNTL_bit] & !gpio_sel) ? {22'b0, !uart_ready, 9'b0}
    : gpio_sel ? gpio_rdata
    : 32'b0;
```

**Replace with:**

```verilog
wire [31:0] IO_rdata =
    (mem_wordaddr[IO_UART_CNTL_bit] & !gpio_sel & !spi_sel) ? {22'b0, !uart_ready, 9'b0}
    : gpio_sel  ? gpio_rdata
    : spi_sel   ? spi_rdata
    : 32'b0;
```

**Note:** The UART CNTL path also gets the `& !spi_sel` guard to prevent misreads when SPI STATUS register is being read.

---

### Change 7 (Simulation only) — Add MISO loopback for BENCH mode

**Add this block** inside `riscv.v`, below the SPI instantiation:

```verilog
`ifdef BENCH
    // MOSI→MISO loopback for simulation only (no physical slave needed)
    assign MISO = MOSI;
`endif
```

This allows simulation with `iverilog -DBENCH` to verify SPI transfers without a physical SPI slave connected.

---

### Verification of All 6 Changes

After making all changes, verify with grep:

```bash
grep -n "spi_master\|spi_sel\|spi_rdata\|SCLK\|MOSI\|MISO\|CS_N" RTL/riscv.v
```

Expected output should show:
- Line with `` `include "spi_master.v" ``
- Lines with `output SCLK`, `output MOSI`, `input MISO`, `output CS_N`
- Lines with `wire spi_rdata`, `wire spi_sel`
- SPI instantiation lines (`.clk(clk)`, `.SCLK(SCLK)` etc.)
- LEDS guard with `& !spi_sel`
- uart_valid guard with `& !spi_sel`
- IO_rdata mux with `spi_sel ? spi_rdata`

---

## Step 4 — Update PCF Pin Assignments

Open `RTL/VSDSquadronFM.pcf` and add the four SPI pin assignments:

### Before

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
```

### After (add 4 lines at the bottom)

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

### Pin Reference (VSDSquadron FM iCE40UP5K-SG48)

| Signal | Pin | IO Bank | Notes |
|--------|-----|---------|-------|
| `SCLK` | 6 | Bank 3 | SPI clock output |
| `MOSI` | 9 | Bank 3 | Master-Out Slave-In |
| `MISO` | 10 | Bank 3 | Master-In Slave-Out |
| `CS_N` | 11 | Bank 3 | Chip Select, active low |
| `CLK` | 35 | Bank 1 | 12 MHz crystal input |

---

## Step 5 — Write the Firmware

Create `Firmware/spi_test.c`:

```c
#include "io.h"

/* -----------------------------------------------------------------------
 * print_hex — sends an 8-digit hex string over UART
 * ----------------------------------------------------------------------- */
static void print_hex(uint32_t v) {
    int i;
    for (i = 7; i >= 0; i--) {
        uint8_t n = (v >> (i * 4)) & 0xF;
        while (IO_IN(IO_UART_CNTL) & 0x100); // wait for UART ready
        IO_OUT(IO_UART_DAT, (n < 10) ? ('0' + n) : ('A' + n - 10));
    }
}

/* -----------------------------------------------------------------------
 * print_str — sends a null-terminated string over UART
 * ----------------------------------------------------------------------- */
static void print_str(const char *s) {
    while (*s) {
        while (IO_IN(IO_UART_CNTL) & 0x100);
        IO_OUT(IO_UART_DAT, *s++);
    }
}

/* -----------------------------------------------------------------------
 * spi_transfer — performs a single 8-bit SPI transfer
 * clkdiv: CLKDIV value (0=6MHz, 4=1.2MHz, 11=500kHz at 12MHz sys clock)
 * tx:     byte to transmit
 * returns: byte received
 * ----------------------------------------------------------------------- */
static uint8_t spi_transfer(uint8_t clkdiv, uint8_t tx) {
    IO_OUT(SPI_TXDATA, tx);
    IO_OUT(SPI_CTRL, ((uint32_t)clkdiv << 8) | 0x3); // START=1, EN=1
    while (!(IO_IN(SPI_STATUS) & 0x2));               // poll DONE
    IO_OUT(SPI_STATUS, 0x2);                          // clear DONE
    return (uint8_t)(IO_IN(SPI_RXDATA) & 0xFF);
}

/* -----------------------------------------------------------------------
 * main
 * ----------------------------------------------------------------------- */
void main() {
    uint8_t rx;

    // Test 1: 0xA5
    rx = spi_transfer(4, 0xA5);
    print_hex(rx);
    if (rx == 0xA5) print_str("PASS\n");

    // Test 2: 0x3C
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

Expected output:
```
Code size: 778 words ( total RAM size: 1536 words )
Occupancy: 50%
max_addr OK
SAVE HEX: spi_test.bram.hex
```

If the Makefile does not have a `spi_test` target, add:

```makefile
spi_test.bram.hex: spi_test.c io.h
	$(MAKE) FIRMWARE=spi_test bram.hex
```

Or compile manually:

```bash
riscv64-unknown-elf-gcc -I ./LIBFEMTOGL -I ./LIBFEMTORV32 -I ./LIBFEMTOC \
    -fno-pic -march=rv32i -mabi=ilp32 -fno-stack-protector -w -Wl,--no-relax \
    -c spi_test.c -o spi_test.o

riscv64-unknown-elf-ld -T bram.ld -m elf32lriscv --nostdlib \
    spi_test.o start.o putchar.o wait.o print.o memcpy.o errno.o perf.o \
    ./libgcc.a -o spi_test.bram.elf

./firmware_words spi_test.bram.elf -ram 6144 -max_addr 6144 -out spi_test.bram.hex

cp spi_test.bram.hex ../RTL/firmware.hex
cp spi_test.bram.hex ../RTL/obj_dir/firmware.hex
echo spi_test.bram.hex > ../RTL/firmware.txt
```

---

## Step 7 — Synthesize and Implement

```bash
cd vsdfpga_labs/basicRISCV/RTL

# Full build (synthesis + P&R + timing + bitstream)
make
```

The Makefile typically runs:

```bash
# 1. Synthesis
yosys -p "synth_ice40 -top SOC -json SOC.json" riscv.v

# 2. Place and Route
nextpnr-ice40 --up5k --package sg48 --json SOC.json \
    --pcf VSDSquadronFM.pcf --asc SOC.asc

# 3. Timing Analysis
icetime -p VSDSquadronFM.pcf -P sg48 -r SOC.timings -d up5k -t SOC.asc

# 4. Pack Bitstream
icepack -s SOC.asc SOC.bin
```

### Expected Synthesis Results

```
Info: Device utilisation:
Info:          ICESTORM_LC:  1136/ 5280    21%
Info:        ICESTORM_RAM:    16/   30    53%
Info:               SB_IO:     8/   96     8%
Info:              SB_GB_IO:   0/    8     0%
...
// Timing estimate: 19.54 MHz (PASS at 12 MHz system clock)
```

---

## Step 8 — Flash to Hardware

### Connect the Board

Connect the VSDSquadron FM via USB. The FTDI chip (0403:6014) provides the programming interface.

### Register the FTDI Device (Linux only)

```bash
sudo bash -c 'echo "0403 6014" | tee /sys/bus/usb-serial/drivers/ftdi_sio/new_id'
```

### Flash

```bash
cd vsdfpga_labs/basicRISCV/RTL
sudo iceprog SOC.bin
```

### Expected Output

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

**Success indicators:**
- `VERIFY OK` — bitstream readback matches written data
- `cdone: high` after flash — FPGA has configured successfully with the new design

---

## Step 9 — Simulation Verification

### 9.1 Standalone IP Testbench (fastest — no SoC needed)

The testbench uses a MOSI→MISO loopback (`assign MISO = MOSI`) to verify the IP in isolation.

```bash
cd vsdfpga_labs/basicRISCV/RTL
iverilog -o spi_tb spi_tb.v spi_master.v
vvp spi_tb
```

Expected output:
```
VCD info: dumpfile spi_tb.vcd opened for output.
TXDATA written: 0xA5
DONE set after 19 polls
RXDATA = 0xa5
PASS: SPI loopback correct
STATUS after clear = 0x00000000
```

View waveforms:
```bash
gtkwave spi_tb.vcd &
```

### 9.2 Full-SoC Simulation

```bash
cd vsdfpga_labs/basicRISCV/RTL
iverilog -DBENCH -o soc_sim riscv.v
timeout 120 vvp soc_sim > /tmp/spi_out.txt 2>&1
```

Decode UART output:
```bash
cat /tmp/spi_out.txt | tr -cd '[:print:]\n' | grep -v "^t=" | cut -c1 | \
    awk 'NR%2==1' | tr -d '\n' | sed 's/PASS/\nPASS\n/g'
echo ""
```

Expected output:
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
| `RXDATA = 0xxx` (unknown) | Registers not reset in spi_master.v | Verify the `if (!resetn)` block initializes ALL registers |
| Simulation runs forever | UART baud rate too high → truncation of START_VALUE | Use `BENCH baud = 200000`, not 1000000 or 9600 |
| RXDATA = 0x00 in SoC sim | MISO floats as X | Add `` `ifdef BENCH assign MISO = MOSI; `endif `` in riscv.v |
| Simulation hangs at polling | Transfer never completing | Check that `busy` and `done` are initialized in reset block |

### Synthesis Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `SB_HFOSC` unsupported | Local nextpnr doesn't support internal oscillator | Use external CLK pin 35; do not use SB_HFOSC |
| `UNKNOWN_FREQUENCY` | femtoPLL given input freq (12) instead of output freq | Use `CPU_FREQ=20` (PLL output); set UART with `clk_freq_hz(12*1000000)` |
| `SB_PLL40_PAD` error | PLL fed from internal oscillator, not IO pad | Connect PLL input directly to CLK pin 35 |
| `ERROR: Multiple drivers` | MISO assigned in both PCF and bench `ifdef` | Ensure `` `ifdef BENCH `` block does not compile for hardware target |

### Hardware Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cdone: low` after flash | Bitstream invalid or FPGA not configured | Check synthesis completed without errors; re-run `make` |
| SCLK not toggling | EN=0 or START not fired | Check io.h offsets; verify CLKDIV in CTRL write |
| RXDATA always 0x00 | MISO pin floating (no slave connected) | Connect a loopback wire between MISO (pin 10) and MOSI (pin 9) for hardware testing |
| Spurious UART output | Missing `& !spi_sel` guard on `uart_valid` | Apply Change 5 in Step 3 above |
| LEDs flashing on SPI writes | Missing `& !spi_sel` on LEDS write path | Apply Change 4 in Step 3 above |

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
    // ── Address decode ──────────────────────────────────────────────────
    wire sel_ctrl   = isIO & (mem_wordaddr[7:0] == 8'd16);
    wire sel_txdata = isIO & (mem_wordaddr[7:0] == 8'd17);
    wire sel_rxdata = isIO & (mem_wordaddr[7:0] == 8'd18);
    wire sel_status = isIO & (mem_wordaddr[7:0] == 8'd19);

    assign spi_sel = sel_ctrl | sel_txdata | sel_rxdata | sel_status;

    // ── Internal registers ───────────────────────────────────────────────
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

    // ── MOSI continuously driven from MSB of shift register ─────────────
    assign MOSI = shift_reg[7];

    // ── Synchronous control logic ────────────────────────────────────────
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
            // Register writes
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

            // SPI transfer engine
            if (busy) begin
                if (clk_cnt < clkdiv) begin
                    clk_cnt <= clk_cnt + 1;
                end else begin
                    clk_cnt <= 0;
                    SCLK    <= ~SCLK;
                    if (!SCLK) begin   // rising edge: sample MISO
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

    // ── Read data mux (combinational) ────────────────────────────────────
    always @(*) begin
        if      (sel_ctrl)   spi_rdata = {16'h0, clkdiv, 6'h0, 1'b0, en};
        else if (sel_rxdata) spi_rdata = {24'h0, rx_data};
        else if (sel_status) spi_rdata = {30'h0, done, busy};
        else                 spi_rdata = 32'h0;
    end
endmodule
```
