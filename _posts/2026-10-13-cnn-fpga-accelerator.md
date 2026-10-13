---
title: "Building a CNN Accelerator on an FPGA from Scratch"
description: "A handwritten-digit neural network quantized to 8-bit integers and rebuilt as custom hardware in SystemVerilog — 98.4% accuracy on MNIST, bit-exact with its software model, and 85.9 µs per image on an Artix-7."
date: 2026-10-13
thumbnail: "/assets/Images/CNN-FPGA/cnn-datapath.png"
image: "/assets/Images/Social/cnn-fpga.jpg"
tags: [FPGA, SystemVerilog, Machine Learning, Quantization, Digital Design, Vivado]
math: true
---

<div class="post-meta-bar" style="display:flex; flex-wrap:wrap; gap:16px; font-size:0.88rem; color:#6b7280; margin-bottom:32px; padding:16px 0; border-bottom:1px solid #e5e7eb;">
  <span><strong>Timeframe:</strong> September – October 2026</span>
  <span><strong>Role:</strong> Solo project</span>
  <span><strong>Type:</strong> Self-directed learning</span>
  <span><strong>Tools:</strong> PyTorch, Vivado, SystemVerilog, Python, Claude (AI assistant), Cmod A7-35T (Artix-7)</span>
  <span><strong>Read time:</strong> ~10 min</span>
</div>

## Introduction

A neural network running on a CPU or GPU is software. I wanted to see what happens when you remove the processor entirely and build the network as hardware: no instructions, no operating system, just multipliers, memories and control logic laid out to match the shape of the computation.

So I trained a small convolutional network to recognize handwritten digits, converted every weight and activation from floating point to 8-bit integers, and built the whole inference path in SystemVerilog on an Artix-7 FPGA. The finished accelerator classifies a digit in **8,590 clock cycles — 85.9 µs at 100 MHz** — using **36 of the chip's 90 DSP slices and about 10% of its logic**, and it agrees with its software reference on all 10,000 test images, bit for bit.

**How this one was built.** Unlike my other projects, I worked through this one with Claude, an AI assistant. The design decisions were shared: we went back and forth on the architecture, the quantization scheme and the hardware budget. All the code — the SystemVerilog, the training and quantization scripts, the reference model and the host tooling — was written by the AI from those decisions, and debugging was mostly AI-assisted too. What was mine: the choice of project, half of the design calls, the review of what came back, the bring-up on real hardware, the bug that made the board classify at 10% accuracy, and this write-up. I'm flagging it because the distinction matters when you read the rest of this page.

## The Network

The architecture follows LeNet-5, scaled to fit comfortably in a small FPGA. Two convolution layers with 3×3 filters, each followed by ReLU and 2×2 max-pooling, then a dense layer that maps 400 features to 10 scores, one per digit, and finally argmax.

| # | Layer | Output | Weights |
|---|---|---|---|
| 0 | Input image | 26×26×1 | — |
| 1 | Conv 3×3, 8 filters | 24×24×8 | 72 (+8 bias) |
| 2–3 | ReLU, max-pool 2×2 | 12×12×8 | — |
| 4 | Conv 3×3, 16 filters | 10×10×16 | 1,152 (+16) |
| 5–6 | ReLU, max-pool 2×2 | 5×5×16 | — |
| 7–8 | Flatten, dense | 10 | 4,000 (+10) |
| 9 | Argmax | 4-bit digit | — |

Training in PyTorch took five passes over the 60,000 MNIST training images and reached **98.4%** accuracy on the 10,000 test images.

One detail saved hardware before a single line of Verilog was written. MNIST images are 28×28, but the second max-pool receives an 11×11 map and silently discards its last row and column. Tracing that backwards, the last two rows and columns of the input image only ever influence the data that gets thrown away. **Cropping the image to 26×26 at the input changes nothing in the result** and shrinks every buffer and loop in the design.

## From Floats to Integers

The trained model holds 5,258 parameters in 32-bit floating point: 168,256 bits of storage, and multipliers that an FPGA would rather not build. Quantizing to 8-bit integers cuts the storage to 42,064 bits and turns every multiply into something a DSP slice does natively.

Quantization maps a real number to an integer through a scale and a zero point, $$r = S(q - Z)$$. For weights I used a symmetric range, which forces $$Z = 0$$ and removes a whole term from the hardware; each filter gets its own scale, since one filter's weights may be much smaller than another's. For activations the range comes from measurement: I ran images through the model and recorded the actual minimum and maximum after each ReLU.

Substituting those into a convolution and isolating the integer output gives what the hardware actually computes:

$$q_y = Z_y + \frac{S_w S_x}{S_y}\left(\sum_i q_{w,i}\, q_{x,i} - Z_x \sum_j q_{w,j} + q_b\right)$$

Everything that doesn't depend on the image is folded into constants on the host, so the hardware only accumulates products of 8-bit numbers. The remaining scale factor is a fraction below 1, which is awkward in integer hardware. The standard trick is to turn it into a multiply and a shift: pick $$M_0 = \text{round}(M \cdot 2^{25})$$, multiply, then shift right by 25 with rounding.

That choice of 25 bits isn't arbitrary. It makes $$M_0$$ fit in **17 bits**, and a DSP slice multiplies 25 bits by 18 bits, so the scale goes in one port and the accumulator (which needs at most 22 bits) in the other. **One DSP slice does the entire requantization step.**

The input pixels get the same treatment through a 256-entry lookup table that maps a raw pixel straight to its normalized, quantized value. It's generated in Python and baked into the bitstream at synthesis, so the board never needs the host to prepare an image.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/input-ordering.png" alt="Block diagram: 28x28 pixel image arrives over UART, passes a 256-entry quantization lookup table, then a 621-deep FIFO in block RAM, then a 55-deep shift register built from LUTs">
  <figcaption>The input path — pixels arrive over UART, get quantized by a lookup table, and flow into a line buffer that presents a 3×3 window to the filters</figcaption>
</figure>

## Designing the Datapath

A convolution needs a 3×3 window of pixels, but pixels arrive one at a time in raster order. The classic solution is a line buffer: a shift register holding two full rows plus three pixels. Once it fills, positions 0, 1, 2, 26, 27, 28, 52, 53 and 54 are exactly the 3×3 window, and every shift slides the window one pixel to the right. Xilinx LUTs can be configured as shift registers, so the buffer costs almost no logic.

The two convolution stages differ in cost. Stage 1 has 8 filters that each see one input channel, so a window takes 9 cycles. Stage 2 has 16 filters that each see all 8 channels of the previous stage: 72 cycles per window. That imbalance means stage 1 produces results faster than stage 2 consumes them, so a clock-enable signal stalls the earlier stages whenever the path ahead is busy.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/conv-quant-stage1.png" alt="Datapath of the first convolution stage: line buffer and per-filter weight memories feed eight convolution MACs built from DSP slices, whose results are registered and fed through multiplexers into a single quantization MAC">
  <figcaption>The first convolution stage — eight DSP slices accumulate in parallel while a single shared DSP slice requantizes the previous results</figcaption>
</figure>

The DSP budget works out to **36 slices**: 8 for the first convolution and 16 for the second, 10 for the dense layer, and one requantizer per convolution stage. One is enough because requantization takes a single cycle per result while a convolution takes 9 or 72, so the shared slice easily keeps up.

Everything else is chosen to match the memory it needs. Filter weights live in LUTs configured as distributed RAM, biases in flip-flops, the dense layer's 4,000 weights in block RAM arranged so one read fetches one weight for each of the 10 output channels simultaneously, and max-pooling is done as pairs of comparisons with a shift register holding one row of intermediate maxima.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/cnn-datapath.png" alt="Full datapath diagram showing the complete pipeline: input FIFO, line buffer, eight convolution MACs, quantization, max-pool registers and shift registers, second line buffer, sixteen convolution MACs, second quantization and pooling, then ten dense MACs and the argmax index unit">
  <figcaption>The complete datapath, from the input FIFO to the argmax — click to enlarge</figcaption>
</figure>

## Talking to the Board

The host talks to the FPGA over a single UART link with six commands: load the 5,456-byte parameter blob, send a 784-byte image, read back the digit, read the 10 raw scores, read the cycle count, and dump any of four intermediate feature maps captured inside the pipeline.

That last command is the one that made the project debuggable. I also wrote a **bit-exact reference model in NumPy** that reproduces exactly what the hardware does, including the rounding and clamping of every requantization. With taps inside the pipeline and a model that should match them exactly, a wrong answer stops being a mystery: you compare stage by stage, and the first stage that differs is where the bug is.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/board-peripherals.png" alt="The Cmod A7-35T board with annotations pointing out the reset button, the activity LED and the heartbeat LED">
  <figcaption>The Cmod A7-35T, an Artix-7 XC7A35T on a breadboard-friendly module. A heartbeat LED shows the design is alive; an activity LED blinks while images are processed <span style="font-style:normal">(board photo: Digilent)</span></figcaption>
</figure>

## Missing 100 MHz by 79 Picoseconds

The first implementation failed timing at 100 MHz, with a worst slack of **−0.079 ns** across 7 failing endpoints. The critical path ran from the output of the second stage's requantizer to the dense layer's DSP inputs, and Vivado's report showed the time going into routing rather than logic.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/critical-path.png" alt="Schematic of the critical timing path: quantization MAC output through demultiplexer, registers, multiplexer, max units, shift register and into the dense layer's convolution MAC">
  <figcaption>The critical path — from the requantizer, through pooling, to the dense layer's multipliers</figcaption>
</figure>

The fix was to register the requantizer's clamped output, cutting the long combinational path in two at the cost of a handful of flip-flops and one cycle of latency per stage. After that the design closed timing with **+0.898 ns** of slack to spare.

The finished design uses a small fraction of the chip:

| Resource | Used | Available |
|---|---|---|
| LUTs | 1,992 | 20,800 |
| Flip-flops | 2,512 | 41,600 |
| Block RAM | 5.5 | 50 |
| DSP slices | 36 | 90 |

Vivado estimates total power at **0.253 W**, most of it in the clock generator rather than the arithmetic.

<figure style="max-width:420px; margin-left:auto; margin-right:auto;">
  <img src="/Blog/assets/Images/CNN-FPGA/device-view.png" alt="Vivado device view of the Artix-7 die with this design's cells highlighted in yellow, forming two clusters in an otherwise empty fabric">
  <figcaption>The implemented design on the die — the highlighted cells are the whole accelerator, one cluster per convolution stage, in a mostly empty chip</figcaption>
</figure>

## Bring-Up: 10% Accuracy

With the bitstream loaded, the first run on real hardware classified 200 images with **10.5% accuracy** — about what you'd get by guessing. The same images through the reference model gave 99.5%, and the two disagreed on 179 of 200.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/board-first-run.png" alt="Console output showing hardware accuracy 0.1050, integer model accuracy 0.9950, and 179 images where the hardware disagrees with the model">
  <figcaption>First run on the board — the hardware agrees with the model on 21 images out of 200</figcaption>
</figure>

A disagreement that large is rarely a subtle arithmetic bug; it looks like the hardware is computing with the wrong numbers entirely. Instead of probing the pipeline, I ran the consistency check that compares the parameter file sent to the board against the quantization parameters the reference model uses. They didn't match: **the parameter blob had been exported by an older version of the quantization script.** The hardware was faithfully computing with stale weights.

After re-running the export and reloading, the same 200 images gave **99.5%**, with zero disagreements against the model.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/board-after-fix.png" alt="Console output after re-exporting the parameters: hardware accuracy 0.9950, zero images where hardware differs from the integer model, bit-exact digits">
  <figcaption>After re-exporting the parameters — 199 of 200 correct, and bit-exact agreement with the model</figcaption>
</figure>

Dumping the intermediate feature maps confirmed it properly: every stage inside the FPGA matches the reference model element for element.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/bit-exact-streams.png" alt="Console output comparing hardware and reference model stage by stage: conv1, pool1, conv2, pool2 and scores all reported identical, concluding the hardware is bit-exact for this image">
  <figcaption>Stage-by-stage comparison — the convolution, pooling and score stages are identical to the software model</figcaption>
</figure>

## Results

Running the complete 10,000-image MNIST test set through the board:

- **98.41% accuracy**, exactly matching the integer reference model, with **0 images** classified differently.
- **8,590 cycles per image — 85.9 µs** at 100 MHz.
- **10.4 images per second**, limited entirely by the UART link, which must carry 785 bytes per image.

<figure>
  <img src="/Blog/assets/Images/CNN-FPGA/full-test-set-run.png" alt="Console output of the full test set run: 10000 of 10000 images, hardware accuracy 0.9841, zero images differing from the integer model, 8590 cycles inference latency, 10.4 images per second throughput">
  <figcaption>The full test set on hardware — identical accuracy to the software model, 85.9 µs per image</figcaption>
</figure>

The quantization cost essentially nothing in accuracy: the float model scored 98.4%, and the 8-bit hardware scores 98.41%. Inference takes 86 µs while sending the image takes 96 ms, so the accelerator spends over 99.9% of its time waiting for pixels.

## Lessons Learned

- **Design the number format around the hardware you have.** Choosing a 25-bit shift wasn't a numerical decision, it was a hardware one: it made the multiplier constant fit the DSP slice's 18-bit port and the accumulator fit the 25-bit one, so requantization costs exactly one DSP slice instead of a pile of logic.
- **A bit-exact software model turns debugging into bisection.** Building the integer model and the debug taps felt like a detour at the time. They turned a wrong answer from the board into a stage-by-stage comparison with a definite answer.
- **Check that the data is right before suspecting the logic.** The board behaved like it had a deep arithmetic bug, and the real cause was a parameter file from an older export. Verifying what you loaded is cheaper than probing what you built.
- **Timing failures are often routing, not logic.** Missing 100 MHz by 79 ps looked alarming until the report showed the delay was in the wires. One register in the right place bought 0.9 ns.
- **Know what actually limits performance.** The compute is 86 µs; the link that feeds it is 96 ms. Any further work on the datapath would be invisible next to the cost of getting an image onto the chip.

<div style="margin-top:40px; padding-top:20px; border-top:1px solid #e5e7eb;">
  <p style="font-size:0.85rem; color:#6b7280; margin-bottom:12px; font-weight:600; text-transform:uppercase; letter-spacing:0.05em;">Skills & Technologies</p>
  <div style="display:flex; flex-wrap:wrap; gap:8px;">
    <span class="tag">SystemVerilog</span>
    <span class="tag">FPGA Design</span>
    <span class="tag">CNN Inference Hardware</span>
    <span class="tag">Post-Training Quantization</span>
    <span class="tag">Fixed-Point Arithmetic</span>
    <span class="tag">DSP48 / BRAM / SRL</span>
    <span class="tag">Timing Closure</span>
    <span class="tag">Hardware/Software Co-Design</span>
    <span class="tag">UART Protocol</span>
    <span class="tag">PyTorch</span>
    <span class="tag">NumPy</span>
    <span class="tag">Python</span>
    <span class="tag">Vivado</span>
    <span class="tag">Artix-7</span>
  </div>
</div>
