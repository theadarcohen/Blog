---
title: "Building a MIPS Processor on an FPGA from Scratch"
description: "A 32-bit pipelined MIPS processor in Verilog — from basic gates through a 5-stage pipeline with forwarding, to I/O integration with an OLED display and keypad — running a custom game on an Artix-7 FPGA at 90 MHz."
date: 2025-9-10
thumbnail: "/assets/Images/FPGA-Processor/final-setup.png"
tags: [FPGA, Verilog, Computer Architecture, Digital Design, Vivado]
---

<div class="post-meta-bar" style="display:flex; flex-wrap:wrap; gap:16px; font-size:0.88rem; color:#6b7280; margin-bottom:32px; padding:16px 0; border-bottom:1px solid #e5e7eb;">
  <span><strong>Timeframe:</strong> September 2025</span>
  <span><strong>Role:</strong> Solo project</span>
  <span><strong>Type:</strong> Self-directed learning</span>
  <span><strong>Tools:</strong> Vivado, Verilog/SystemVerilog, Python, Artix-7 FPGA</span>
  <span><strong>Read time:</strong> ~10 min</span>
</div>

## Introduction

I wanted to understand how a CPU actually works — not at the block-diagram-in-a-textbook level, but at the gate-and-register level where you have to make every design decision yourself. So I designed and built a 32-bit MIPS processor in Verilog, starting from basic arithmetic modules and working up to a fully pipelined architecture with data forwarding and hazard detection. Then I kept going: I integrated an I2C OLED display and a matrix keypad, wrote assembly-level drivers for both peripherals, and ultimately ran a playable game — all on custom hardware running on a 100$ FPGA board.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/fpga-board.png" alt="Artix-7 FPGA evaluation board used for the project">
  <figcaption>The Artix-7 FPGA (XC7A35T) evaluation board — the target platform for the processor</figcaption>
</figure>

## Instruction Set Architecture

The processor implements a custom MIPS-based ISA with three instruction formats: R-type (register-register operations like `add`, `sub`, `slt`, `mult`, `div`, shifts), I-type (immediate operations, loads, stores, branches), and J-type (jumps). The register file contains 32 general-purpose 32-bit registers following the standard MIPS convention, with `$0` hardwired to zero and `$31` used as the return address register.

I also built a **Python assembler** that translates human-readable assembly source files into binary machine code, outputting `.mif` (Memory Initialization File) format that Vivado's block RAM can load directly. The assembler handles labels, comments, and all instruction encodings — turning assembly programs into bitstreams the hardware can execute.

## Single-Cycle Processor

The first working processor was a single-cycle design: one instruction completes per clock cycle, with the entire datapath — fetch, decode, execute, memory access, and write-back — happening combinationally between clock edges.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/basic-processor-architecture.png" alt="Block diagram of the single-cycle MIPS processor architecture showing ALU, register file, memory, and control paths">
  <figcaption>The single-cycle processor architecture — every instruction completes in one clock cycle</figcaption>
</figure>

The design includes an ALU supporting arithmetic, logical, shift, multiplication, and division operations; a dual-port memory (separate instruction and data ports); a register file with two read and one write port; and a controller implemented as a state machine that generates all the mux-select and write-enable signals based on the opcode.

### Timing Analysis and the Case for Pipelining

When I ran Vivado's timing analysis on the single-cycle design at 100 MHz, the result was a **Worst Negative Slack (WNS) of &minus;5 ns** — the critical path couldn't settle in time. The worst offenders were long combinational chains like Instruction Memory &rarr; Register File &rarr; ALU (Multiply) &rarr; Hi Register, with 17 levels of logic. To meet timing at 100 MHz, I used the FPGA's internal PLL (via Vivado's Clocking Wizard IP) to generate a 50 MHz clock — functional, but half the potential performance.

This was the direct motivation for pipelining: by inserting registers between stages, each stage's combinational depth shrinks, and the clock frequency can go up.

## Pipeline Processor

The pipeline splits the datapath into five classic stages — **Instruction Fetch (IF)**, **Instruction Decode (ID)**, **Execute (EX)**, **Memory Access (MEM)**, and **Write-Back (WB)** — with pipeline registers between each pair.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/pipeline-architecture.png" alt="Architecture diagram of the 5-stage pipeline MIPS processor with stage registers">
  <figcaption>The 5-stage pipeline processor architecture with inter-stage registers</figcaption>
</figure>

### Data Forwarding and Hazard Detection

The real complexity of a pipeline lies in handling hazards. A **data hazard** occurs when an instruction depends on the result of a preceding instruction that hasn't written back yet. Without intervention, the pipeline reads stale register values.

I implemented a **forwarding unit** that detects these dependencies and bypasses results directly from the EX/MEM or MEM/WB pipeline registers back to the ALU inputs, avoiding stalls in most cases. For load-use hazards — where a `lw` is immediately followed by an instruction consuming its result — forwarding alone isn't enough, so the hazard detection unit inserts a one-cycle stall (pipeline bubble).

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/forwarding-architecture.png" alt="Processor architecture sketch highlighting the forwarding paths from EX/MEM and MEM/WB stages back to the ALU inputs">
  <figcaption>The forwarding paths — results are bypassed from later pipeline stages back to the ALU inputs to resolve data hazards without stalling</figcaption>
</figure>

**Control hazards** from branches are resolved in the ID stage using a dedicated branch comparator, minimizing the branch penalty to a single flush cycle. The pipeline controller manages all forwarding mux selects, stall signals, and flush logic.

### Timing Results

With the pipeline in place, the timing picture improved dramatically. At 100 MHz, the WNS shrank to only &minus;0.53 ns — a 10&times; improvement over the single-cycle design's &minus;5 ns. Pushing the clock to **90 MHz** gave positive slack across all paths, meaning the design meets timing with margin to spare.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/timing-analysis-90mhz.png" alt="Vivado Design Timing Summary showing positive WNS at 90 MHz clock frequency">
  <figcaption>Timing analysis at 90 MHz — all paths meet timing with positive slack</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/pipeline-simulation.png" alt="Simulation waveform showing pipeline operation with correct instruction execution">
  <figcaption>Pipeline simulation waveform — verifying correct instruction execution with forwarding active</figcaption>
</figure>

## On-Hardware Debugging with the ILA

Simulation only goes so far. To verify the processor was actually executing correctly on silicon, I used Vivado's **Integrated Logic Analyzer (ILA)** — an IP core synthesized into the FPGA fabric that captures internal signals in real time over JTAG without requiring external probes.

I instrumented the design to expose key internal signals: the memory data output (`read_mem_data`), the `halt_state` flag, and the button inputs used for stepping and triggering. After pressing the board's "process" button to reset and run the CPU, I triggered the ILA to scan through memory and dump the contents as a CSV waveform capture, which a Python script then converted back into binary for verification against expected results.

This was invaluable for catching subtle issues that don't show up in simulation — timing interactions with the FPGA's clock distribution, real BRAM behavior, and physical button debouncing — giving confidence that the design worked correctly on actual hardware, not just in a testbench.

## I/O: OLED Display and Matrix Keypad

With a working CPU, the next step was connecting it to the outside world. I designed hardware controllers for two peripherals: an **SSD1306 128&times;32 OLED display** (I2C interface) and a **4&times;4 matrix membrane keypad**.

### OLED Display Controller

The display controller consists of two main subsystems: a **cross-clock domain FIFO** and an **I2C controller**.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/display-controller-block-diagram.png" alt="Block diagram of the display controller architecture showing cross-clock domain FIFO and I2C controller">
  <figcaption>Display controller architecture — a cross-clock domain FIFO bridges the 90 MHz CPU and the ~156 kHz I2C bus</figcaption>
</figure>

The CPU runs at 90 MHz; the I2C bus runs at ~156 kHz. Passing data between these clock domains safely requires a proper **asynchronous FIFO** with Grey code pointer synchronization to prevent metastability. The write pointer (CPU clock domain) and read pointer (I2C clock domain) are each converted to Grey code, passed through a two-stage synchronizer into the opposite domain, and then converted back to binary for full/empty detection.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/i2c-state-machine.png" alt="State machine diagram of the I2C controller showing IDLE, START, WORK, ACK, STOP, and ACK_N states">
  <figcaption>The I2C controller state machine — handles start/stop conditions, byte transmission, and ACK/NACK detection</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/oled-display.jpeg" alt="Photo of the SSD1306 OLED display module on its breakout board">
  <figcaption>The SSD1306 OLED display on its breakout board — 128&times;32 pixels, driven over I2C</figcaption>
</figure>

### Matrix Keypad Controller

The keypad controller scans columns sequentially and reads back rows to detect which keys are pressed. It includes debouncing logic (820 &mu;s debounce window at 10 MHz) and outputs a 16-bit `keys_pressed` register that the CPU can read through a memory-mapped I/O register.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/keypad-structure.png" alt="Diagram showing the matrix structure of the 4x4 membrane keypad with row and column lines">
  <figcaption>The 4&times;4 matrix keypad structure — columns are scanned sequentially, rows are read to detect presses</figcaption>
</figure>

## System Integration

The final system ties the pipeline CPU, display controller, and keypad controller together through **memory-mapped I/O registers**. The CPU accesses the display's FIFO data register at address 1024, a display control register at 1025, and a display status register at 1026. The keypad's `keys_pressed` register is mapped to address 1027.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/integration-schematic.png" alt="Integration schematic showing the pipeline CPU connected to the display and keypad controllers through memory-mapped I/O">
  <figcaption>System integration — the pipeline CPU communicates with I/O controllers through memory-mapped registers</figcaption>
</figure>

The Clocking Wizard IP generates two clocks from the 100 MHz board oscillator: 90 MHz for the CPU and 10 MHz for the I/O controllers (further divided down for the I2C bus).

## Assembly Software: Drivers and a Game

With the hardware complete, I wrote all the software in MIPS assembly — there is no C compiler, no OS, no abstraction layer. Every interaction with the display and keypad is manual register manipulation.

### Display Drivers

I built a library of assembly functions following a calling convention (parameters in `$4`–`$7`, return value in `$2`, stack pointer in `$29`):

- **`WriteDisplay`** — sends an arbitrary byte sequence from memory to the display via the FIFO, with flow control (waits when FIFO is full) and error handling (aborts on I2C NACK). It extracts individual bytes from 32-bit memory words using shifts.
- **`InitializeDisplay`** — sends the SSD1306 initialization sequence: Display Off, Charge Pump On, Display On.
- **`InitializeDisplayRAM`** — clears the entire display RAM by writing zeros to all columns across all pages, using Page Addressing mode.
- **`SetColumnPage`** — sets the current write position on the display (column and page address) for subsequent data writes.
- **`EntireDisplay`** — turns all pixels on or off regardless of RAM content (useful for testing).
- **`AddressingMode`** — switches between Horizontal and Page addressing modes.

### CatcherGame

The culmination of the project is a playable game written entirely in assembly. A target appears at the top of the OLED screen and falls downward page by page. The player controls a catcher at the bottom using the keypad (keys '5' and '6' for left/right movement). If the catcher is in the right column when the target reaches the bottom, it scores a catch and a new target spawns at a random column (generated by a **Linear Congruential Generator** implemented in assembly using the hardware multiplier). If the catcher misses, the program halts.

The game involves coordinated timing loops, polling the keypad register, updating the display through the driver functions, and managing sprite positions — all in hand-written assembly running on custom hardware.

<figure>
  <img src="/Blog/assets/Images/FPGA-Processor/final-setup.png" alt="Photos of the final hardware setup showing the FPGA board connected to the OLED display and matrix keypad">
  <figcaption>The final setup — FPGA board with the OLED display and matrix keypad connected, running the CatcherGame</figcaption>
</figure>

## Lessons Learned

- **Pipelining is the single biggest architectural win.** The jump from a single-cycle design (50 MHz, limited by the worst-case combinational path) to a 5-stage pipeline (90 MHz, with near-ideal throughput) demonstrated concretely why every modern processor is pipelined. The added complexity of forwarding and hazard detection is the price of admission.
- **Clock domain crossings demand respect.** Getting data safely between the 90 MHz CPU and the ~156 kHz I2C bus required a proper asynchronous FIFO with Grey code synchronizers. Any shortcut here leads to intermittent, hard-to-debug metastability failures.
- **Timing analysis drives architecture.** The single-cycle design's &minus;5 ns WNS wasn't just a number — it told me exactly which paths were too long and motivated the pipeline split. Learning to read Vivado's timing reports and trace critical paths was as valuable as designing the logic itself.
- **Writing software for your own hardware closes the loop.** Building the assembler, writing the drivers, and running a game on a processor I designed gave me an understanding of the hardware-software interface that no textbook could.

<div style="margin-top:40px; padding-top:20px; border-top:1px solid #e5e7eb;">
  <p style="font-size:0.85rem; color:#6b7280; margin-bottom:12px; font-weight:600; text-transform:uppercase; letter-spacing:0.05em;">Skills & Technologies</p>
  <div style="display:flex; flex-wrap:wrap; gap:8px;">
    <span class="tag">Verilog / SystemVerilog</span>
    <span class="tag">FPGA Design</span>
    <span class="tag">Computer Architecture</span>
    <span class="tag">Pipeline Design</span>
    <span class="tag">Data Forwarding</span>
    <span class="tag">Hazard Detection</span>
    <span class="tag">I2C Protocol</span>
    <span class="tag">Clock Domain Crossing</span>
    <span class="tag">Vivado</span>
    <span class="tag">Timing Analysis</span>
    <span class="tag">Python (Assembler)</span>
    <span class="tag">MIPS Assembly</span>
    <span class="tag">Memory-Mapped I/O</span>
    <span class="tag">Artix-7 FPGA</span>
  </div>
</div>
