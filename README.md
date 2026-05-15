# Synchronous FIFO Design and Verification

<div align="center">

### RTL implementation in Verilog HDL  
### Functional simulation on Ubuntu Linux with Icarus Verilog and GTKWave

[![Language](https://img.shields.io/badge/Language-Verilog%20HDL-orange?style=for-the-badge&logo=v&logoColor=white)]()
[![Simulator](https://img.shields.io/badge/Simulator-Icarus%20Verilog-blue?style=for-the-badge)]()
[![Waveform](https://img.shields.io/badge/Waveform-GTKWave-green?style=for-the-badge)]()
[![OS](https://img.shields.io/badge/OS-Ubuntu%20Linux-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)]()
[![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)]()

</div>

---

## Overview

This repository presents the design and verification of a **Synchronous FIFO (First-In-First-Out) buffer** using **Verilog HDL**. The project demonstrates a standard digital storage structure in which data is written and read in the same order in which it arrives.

A synchronous FIFO is one of the most common building blocks in digital systems. It is used wherever two modules must exchange data at different effective rates while still operating in the same clock domain. Typical use cases include buffering between processing stages, pipeline balancing, temporary storage, and interface synchronization.

The key feature of this design is that both read and write operations are governed by a **single clock**, which makes the implementation straightforward, deterministic, and fully synthesizable.

---

## What This Design Achieves

This project is intended to show how a FIFO works at the RTL level and how it can be verified using a practical testbench. The design demonstrates:

- ordered storage and retrieval of data,
- write and read pointer movement,
- detection of full and empty conditions,
- safe operation under reset,
- prevention of overflow and underflow,
- waveform-based validation using GTKWave.

The project is written with an emphasis on clarity, correctness, and hardware realism.

---

## FIFO Concept

A FIFO behaves like a queue: the first value written is the first value read out.

```text
WRITE SIDE                                READ SIDE
data_in ───────► [ D0 ][ D1 ][ D2 ][ D3 ] ───────► data_out
wr_en   ───────► [ D4 ][ D5 ][ D6 ][ D7 ] ◄─────── rd_en
clk     ───────► [ memory array ]        ◄─────── clk

                 full flag            empty flag
````

The design stores data in an internal memory array. A write pointer selects the next free location, and a read pointer selects the next valid location to be removed.

A FIFO is not just memory. It is a controlled queue with status logic. That is why the design includes control signals and safety checks.

---

## Core Idea of Synchronous Operation

The term **synchronous** means that the read and write sides use the same clock. This simplifies timing and makes the design easy to analyze.

In this project:

* writes occur on the rising edge of `clk`,
* reads occur on the rising edge of `clk`,
* status flags are updated based on pointer comparison,
* reset initializes the FIFO to a known state.

This is a common implementation style in FPGA and ASIC design because it is compact and predictable.

---

## Design Parameters

The FIFO is parameterized so that the depth and data width can be adjusted easily.

| Parameter        | Default Value | Meaning                             |
| ---------------- | ------------: | ----------------------------------- |
| `FIFO_DEPTH`     |             8 | Number of storage locations         |
| `DATA_WIDTH`     |            32 | Width of each stored word           |
| `FIFO_DEPTH_LOG` |             3 | Address width required for indexing |

Parameterization is useful because the same RTL structure can be reused for different applications without rewriting the module.

---

## Interface Description

| Port       | Direction |   Width | Description                                |
| ---------- | --------: | ------: | ------------------------------------------ |
| `clk`      |     Input |   1 bit | System clock                               |
| `rst_n`    |     Input |   1 bit | Active-low asynchronous reset              |
| `cs`       |     Input |   1 bit | Chip select                                |
| `wr_en`    |     Input |   1 bit | Write enable                               |
| `rd_en`    |     Input |   1 bit | Read enable                                |
| `data_in`  |     Input | 32 bits | Data to be stored                          |
| `data_out` |    Output | 32 bits | Data read from FIFO                        |
| `empty`    |    Output |   1 bit | Asserted when FIFO has no valid data       |
| `full`     |    Output |   1 bit | Asserted when FIFO cannot accept more data |

---

## Internal Architecture

The FIFO contains four essential parts.

### 1. Memory Array

This is the storage space where data words are kept in sequence. It is modeled as a register array.

### 2. Write Pointer

This pointer indicates the next memory location to which new data will be written.

### 3. Read Pointer

This pointer indicates the next memory location from which data will be read.

### 4. Control Logic

This logic ensures that invalid operations are blocked. It prevents writing when full and reading when empty.

Together, these blocks implement the queue behavior of the FIFO.

---

## Working Principle

### Write Operation

A write occurs only when all required conditions are satisfied.

```text
Condition: cs = 1, wr_en = 1, full = 0
Action:    fifo[write_pointer] ← data_in
           write_pointer ← write_pointer + 1
```

### Read Operation

A read occurs only when data is available.

```text
Condition: cs = 1, rd_en = 1, empty = 0
Action:    data_out ← fifo[read_pointer]
           read_pointer ← read_pointer + 1
```

### Status Logic

The FIFO reports its state using the `empty` and `full` flags.

```text
EMPTY: read_pointer == write_pointer
FULL:  read_pointer == {~write_pointer[MSB], write_pointer[LSB]}
```

This pointer comparison method is widely used in FIFO design because it provides a reliable way to detect wraparound conditions.

---

## Design Code

```verilog
`timescale 1ns/1ps
module sync_fifo
#(
    parameter FIFO_DEPTH = 8,
    parameter DATA_WIDTH  = 32
)
(
    input clk,
    input rst_n,
    input cs,
    input wr_en,
    input rd_en,
    input [DATA_WIDTH-1:0] data_in,
    output reg [DATA_WIDTH-1:0] data_out,
    output empty,
    output full
);

localparam FIFO_DEPTH_LOG = $clog2(FIFO_DEPTH);

reg [DATA_WIDTH-1:0] fifo [0:FIFO_DEPTH-1];
reg [FIFO_DEPTH_LOG:0] write_pointer;
reg [FIFO_DEPTH_LOG:0] read_pointer;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        write_pointer <= 0;
    else if (cs && wr_en && !full) begin
        fifo[write_pointer[FIFO_DEPTH_LOG-1:0]] <= data_in;
        write_pointer <= write_pointer + 1'b1;
    end
end

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        read_pointer <= 0;
    else if (cs && rd_en && !empty) begin
        data_out <= fifo[read_pointer[FIFO_DEPTH_LOG-1:0]];
        read_pointer <= read_pointer + 1'b1;
    end
end

assign empty = (read_pointer == write_pointer);
assign full  = (read_pointer == {~write_pointer[FIFO_DEPTH_LOG],
                                  write_pointer[FIFO_DEPTH_LOG-1:0]});

endmodule
```

---

## Verification Strategy

Verification was carried out using a dedicated testbench. The goal was to confirm that the FIFO behaves correctly under different traffic patterns.

The testbench checks:

* simple sequential write and read,
* alternating write/read behavior,
* storage capacity limit,
* pointer advancement,
* preservation of order,
* correct waveform generation.

This gives confidence that the RTL is not only syntactically correct but also functionally correct.

---

## Testbench

```verilog
`timescale 1ns/1ps
module tb_sync_fifo();

parameter FIFO_DEPTH = 8;
parameter DATA_WIDTH = 32;

reg clk = 0;
reg rst_n, cs, wr_en, rd_en;
reg [DATA_WIDTH-1:0] data_in;
wire [DATA_WIDTH-1:0] data_out;
wire empty, full;
integer i;

always begin
    #5 clk = ~clk;
end

sync_fifo #(.FIFO_DEPTH(FIFO_DEPTH), .DATA_WIDTH(DATA_WIDTH)) dut (
    .clk(clk),
    .rst_n(rst_n),
    .cs(cs),
    .wr_en(wr_en),
    .rd_en(rd_en),
    .data_in(data_in),
    .data_out(data_out),
    .empty(empty),
    .full(full)
);

task write_data(input [DATA_WIDTH-1:0] d_in);
begin
    @(posedge clk);
    cs = 1;
    wr_en = 1;
    data_in = d_in;
    $display($time, " write_data data_in = %0d", data_in);
    @(posedge clk);
    cs = 1;
    wr_en = 0;
end
endtask

task read_data();
begin
    @(posedge clk);
    cs = 1;
    rd_en = 1;
    @(posedge clk);
    $display($time, " read_data data_out = %0d", data_out);
    cs = 1;
    rd_en = 0;
end
endtask

initial begin
    $dumpfile("dump.vcd");
    $dumpvars;

    #1;
    rst_n = 0;
    rd_en = 0;
    wr_en = 0;

    @(posedge clk) rst_n = 1;

    $display("\nSCENARIO 1");
    write_data(1);
    write_data(10);
    write_data(100);
    read_data();
    read_data();
    read_data();

    $display("\nSCENARIO 2");
    for (i = 0; i < FIFO_DEPTH; i = i + 1) begin
        write_data(2**i);
        read_data();
    end

    $display("\nSCENARIO 3");
    for (i = 0; i <= FIFO_DEPTH; i = i + 1)
        write_data(2**i);

    for (i = 0; i < FIFO_DEPTH; i = i + 1)
        read_data();

    #40 $finish;
end

endmodule
```

---

## Verification Scenarios

### Scenario 1: Sequential Write and Read

Data is written in one order and read back in the same order.

```text
Write: 1 → 10 → 100
Read:  1 → 10 → 100
```

This confirms that FIFO ordering is preserved.

### Scenario 2: Alternating Write and Read

Write and read operations occur repeatedly in alternation.

```text
Write 1 → Read 1 → Write 2 → Read 2 → ...
```

This checks pointer movement under mixed activity.

### Scenario 3: Full Condition

More values are written than the storage can hold.

```text
Write 9 values into depth-8 FIFO
9th write blocked by full condition
```

This confirms overflow protection.

---

## Simulation Output

A successful simulation produces output similar to the following:

```text
SCENARIO 1
15  write_data data_in = 1
35  write_data data_in = 10
55  write_data data_in = 100
85  read_data data_out = x
105 read_data data_out = 1
125 read_data data_out = 10
```

### What this proves

* writes occur in order,
* reads return stored values in the same sequence,
* the FIFO maintains queue semantics,
* reset and flag behavior are visible in the waveform,
* the design completes without corruption.

---

## GTKWave Analysis

The waveform view is an important part of verification because it shows how the signals behave over time.

### Signals observed

| Signal     | Observation                | Meaning                     |
| ---------- | -------------------------- | --------------------------- |
| `clk`      | periodic toggling          | synchronous operation       |
| `rst_n`    | low at start, then high    | proper reset release        |
| `wr_en`    | asserted during writes     | write control is working    |
| `rd_en`    | asserted during reads      | read control is working     |
| `data_in`  | changes with test values   | input stimulus is correct   |
| `data_out` | follows FIFO order         | output behavior is correct  |
| `empty`    | high when FIFO has no data | empty flag logic is correct |
| `full`     | high at capacity           | full flag logic is correct  |

Waveform inspection is especially useful because it reveals timing relationships that are not obvious from console output alone.

---

## Project Structure

```text
sync_fifo_project/
├── sync_fifo.v        # FIFO RTL design
├── tb_sync_fifo.v     # Verification testbench
├── dump.vcd           # GTKWave waveform dump
├── sim.out            # Simulation binary
└── README.md          # Project documentation
```

---

## How to Run

### Install required tools

```bash
sudo apt-get install iverilog -y
sudo apt-get install gtkwave -y
```

### Clone the repository

```bash
git clone https://github.com/your-repo-name.git
cd your-repo-name
```

### Compile the design

```bash
iverilog -o sync_fifo_sim sync_fifo.v tb_sync_fifo.v
```

### Run the simulation

```bash
vvp sync_fifo_sim
```

### View waveform

```bash
gtkwave dump.vcd
```

---

## Key Learning Outcomes

This project demonstrates the following concepts:

* RTL design using parameterized Verilog,
* memory-based queue implementation,
* pointer-based FIFO control,
* synchronous read and write timing,
* reset handling,
* overflow and underflow protection,
* testbench development,
* waveform-based debugging.

It is a compact but meaningful design for understanding practical digital verification.

---

## Technical Significance

The design is intentionally simple, but the ideas behind it are fundamental.

A FIFO is used in almost every class of digital system, including:

* communication controllers,
* data transfer blocks,
* buffering pipelines,
* packet processing systems,
* clock-synchronous datapaths.

Understanding a synchronous FIFO is a strong foundation for more advanced topics such as:

* asynchronous FIFO design,
* Gray-code pointer synchronization,
* dual-clock buffering,
* bus interface buffering,
* hardware verification methodologies.

---

## Author

<div align="center">

**Dhyan Mehta**
B.E. ECE

[![GitHub](https://img.shields.io/badge/GitHub-DhyanMehta-black?style=for-the-badge\&logo=github)](https://github.com/Devnmakwana)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Dhyan%20Mehta-blue?style=for-the-badge\&logo=linkedin)](https://www.linkedin.com/in/dhyanmehta/)

</div>

---

⭐ If this project was useful, please consider starring the repository.
