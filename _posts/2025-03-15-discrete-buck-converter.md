---
title: "Designing a Discrete Buck Converter from Scratch"
description: "A fully discrete 5V-to-3V buck converter — from theory and simulation through PCB layout to hardware testing — with no off-the-shelf controller IC."
date: 2025-03-15
thumbnail: "/assets/Images/Buck-Converter/assembled-board.png"
image: "/assets/Images/Social/buck-converter.jpg"
tags: [Power Electronics, Analog Design, PCB Design, LTspice, KiCad]
---

<div class="post-meta-bar" style="display:flex; flex-wrap:wrap; gap:16px; font-size:0.88rem; color:#6b7280; margin-bottom:32px; padding:16px 0; border-bottom:1px solid #e5e7eb;">
  <span><strong>Timeframe:</strong> March – October 2025</span>
  <span><strong>Role:</strong> Solo project</span>
  <span><strong>Type:</strong> Self-directed learning</span>
  <span><strong>Tools:</strong> LTspice, KiCad, JLCPCB, Analog Discovery 3</span>
  <span><strong>Read time:</strong> ~8 min</span>
</div>

## Introduction

Most buck converter designs rely on an integrated controller IC that hides the interesting engineering inside a black box. I wanted to understand every stage of a switching power supply — the control loop, the PWM generation, the gate driving, the soft-start sequencing — so I designed one entirely from discrete components.

The goal: convert a 5 V input to a regulated 3 V output at 100 mA, switching at 500 kHz, with proper feedback compensation, soft-start, and a custom four-layer PCB. This post walks through the full design process, from first principles to oscilloscope captures of the finished board.

## Theory and Background

A buck converter works by rapidly switching a DC source on and off, then filtering the resulting square wave through an LC low-pass filter to produce a lower, steady DC output. The duty cycle D of the switch directly sets the output voltage: V<sub>out</sub> = D &times; V<sub>in</sub>.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/switching-converter-concept.png" alt="Basic switching converter structure showing a switch, load, and PWM waveform with the relationship Vo(avg) = (ton/T) x Vs">
  <figcaption>Basic switching converter concept — the duty cycle of the switch determines the average output voltage</figcaption>
</figure>

In a practical buck circuit, a MOSFET replaces the ideal switch, a diode (or second MOSFET) provides a freewheeling path for inductor current when the switch is off, and an output capacitor smooths the remaining ripple. The LC filter's corner frequency must sit well below the switching frequency to attenuate the harmonics of the square wave — ideally at least a decade below.

Choosing component values involves a chain of trade-offs. A larger inductor reduces current ripple but limits transient response. A larger output capacitor improves voltage ripple but its Equivalent Series Resistance (ESR) introduces its own ripple component. I targeted an inductor ripple current of roughly 10% of the full load current as a starting point, which led to a 47 &mu;H inductor and a 10 &mu;F output capacitor for my 500 kHz, 100 mA design.

## Simulation and Iterative Design

I developed the circuit through four successive topologies in LTspice, each one addressing limitations uncovered in the previous iteration.

**Topology 1** was a minimal proof of concept — a comparator-based feedback loop directly driving a PMOS/NMOS pair. It worked, but produced enormous current ripple (1.7 A peak) and an unacceptably large output voltage ripple of 1.2 V. The switching frequency was only 17 kHz, set by the comparator's own oscillation rather than a dedicated oscillator.

**Topology 3** introduced a sawtooth wave generator running at 500 kHz and a Type 3 error amplifier for loop compensation. The Type 3 compensator adds two pole-zero pairs to the loop gain, which cancel the complex poles from the LC filter and provide adequate phase margin for stability. I calculated the compensation network values analytically — placing the two zeros at the LC resonant frequency and the poles at the ESR zero and half the switching frequency — then verified in simulation.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/topology-3-schematic.png" alt="LTspice schematic of Topology 3 showing sawtooth generator, PWM comparator, Type 3 error amplifier, and PMOS/NMOS power stage">
  <figcaption>Topology 3 — sawtooth generator, PWM comparator, and Type 3 error amplifier driving PMOS/NMOS switches</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/topology-3-simulation.png" alt="Simulation waveforms showing Vsw square wave at 500 kHz, triangular inductor current around 100 mA, and stable 3V output with millivolt ripple">
  <figcaption>Topology 3 simulation results — <span style="color:#4444ff;">Blue</span>: V<sub>sw</sub>, <span style="color:#ff3333;">Red</span>: inductor current, <span style="color:#33cc33;">Green</span>: output voltage at 3 V with low ripple</figcaption>
</figure>

Compared to Topology 1, the improvement was dramatic: the switching frequency was now a controlled 500 kHz, the inductor current ripple dropped to ~44 mA, and the output held steady at 3.0 V with millivolt-level ripple. Load regulation tests showed that Topology 3 recovered from a step load change roughly five times faster than the simpler Type 1 compensator, thanks to its wider loop bandwidth.

**Topology 4** — the final design — added everything needed for a real circuit: a Schottky diode instead of the low-side NMOS (simplifying the gate drive), a totem-pole BJT gate driver to provide sufficient gate charge current without overloading the comparator, a dual soft-start switch with sequenced power rails (Vdd1 then Vdd2), a Zener-based voltage reference, and input decoupling capacitors.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/topology-4-full-circuit.png" alt="Complete LTspice schematic of Topology 4 showing all subsystems: soft-start switches, reference voltages, sawtooth generator, error amplifier, gate driver, and power stage">
  <figcaption>The final Topology 4 — complete system with soft-start, reference generation, gate driver, and compensated feedback loop</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/topology-4-simulation.png" alt="Simulation showing Vdd, Vdd2, Vcontrol, Vout, and supply current during enable/disable cycle">
  <figcaption>Full enable/disable simulation — <span style="color:#ff3333;">Red</span>: supply current, <span style="color:#ff44ff;">Pink</span>: V<sub>out</sub>, <span style="color:#00cccc;">Cyan</span>: Enable, <span style="color:#3333ff;">Blue</span>: Vdd2, <span style="color:#33cc33;">Green</span>: Vdd</figcaption>
</figure>

The simulation confirmed clean power-up and power-down behavior. Vdd1 ramps first, allowing the control circuitry to stabilize. Vdd2 follows with a controlled delay, only then connecting the power MOSFET and starting the switching action. At turn-off, the sequence reverses.

## PCB Design and Implementation

I translated the schematic into KiCad and designed a four-layer PCB (Signal–Ground–Ground–Signal) manufactured by JLCPCB. Component selection was driven by the simulation: an IRLML6402 PMOS (low R<sub>DS(on)</sub>, low gate charge), MMDT2227 complementary BJTs for the totem-pole driver, a CUS10S30 Schottky diode, OPA2365 op-amps, and TLV3502 comparators — all rail-to-rail, all within the 4–5 V supply range.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/kicad-schematic.png" alt="KiCad schematic showing the buck loop section with sawtooth generator, comparator, totem-pole driver, and Type 3 error amplifier with real component designators">
  <figcaption>KiCad schematic of the buck loop and sawtooth generator section, with real component values and part numbers</figcaption>
</figure>

Layout decisions were guided by signal integrity concerns. I used 0.5 mm traces for high-current paths and copper pours around the inductor area to minimize parasitic resistance. Decoupling capacitors were placed as close as possible to IC supply pins. The sawtooth generator was kept away from the inductor to reduce EMI coupling. Stitching vias throughout the board tied the two ground planes together, minimizing cross-talk from energy propagating between layers.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/pcb-3d-view.png" alt="3D rendering of the assembled PCB showing component placement, mounting holes, and debug headers">
  <figcaption>3D rendering of the final PCB layout — debug headers for Vout, Vsw, Vcontrol, and Vsupply are visible at the edges</figcaption>
</figure>

## Testing and Results

I tested the board using an Analog Discovery 3, which served as both the power supply and the oscilloscope.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/board-test-setup.jpeg" alt="Photo of the assembled PCB with probe clips connected to test points, wired to an Analog Discovery 3">
  <figcaption>The assembled board during testing — probe clips connected to Vsw, Vout, and ground, powered by the Analog Discovery 3</figcaption>
</figure>

**Initial power-on at 4.8 V** revealed that the sawtooth generator was not oscillating — its output was stuck at the positive rail. The comparator input threshold was designed for a 5 V supply, so at 4.8 V the trip point wasn't being reached. Increasing the supply to 5 V immediately brought the sawtooth generator to life, and the switching frequency measured 523 kHz — very close to the 500 kHz target.

At 5 V, the output regulated to 3.28 V (versus the 3.0 V target). I traced the offset to the Zener reference voltage drifting under thermal load — the diode's dynamic impedance of ~120 mV/mA made the reference sensitive to current variations. The duty cycle measured 0.68, and V<sub>sw</sub> showed clean switching transitions.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/switching-waveforms-5v.png" alt="Oscilloscope capture showing PWM comparator output and totem-pole input with clean 523 kHz switching at 5V supply">
  <figcaption>Clean switching at 5 V supply — <span style="color:#cc7700;">Orange</span>: PWM comparator output, <span style="color:#3333ff;">Blue</span>: totem-pole input, switching at 523 kHz</figcaption>
</figure>

**Soft-start debugging** uncovered a hardware error: one Schottky diode (D200) was placed with reversed polarity on the board, which allowed the gate of the soft-start MOSFET to discharge too quickly, bypassing the RC time constant entirely. After resoldering the diode, the soft-start worked as designed — Vdd1 ramped smoothly over ~2 ms, and Vdd2 followed with a controlled delay.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/soft-start-vdd1.png" alt="Oscilloscope capture showing Vdd1 ramping with soft-start after diode fix, compared to the fast enable signal">
  <figcaption>Soft-start working correctly after fixing D200 — <span style="color:#cc7700;">Orange</span>: Enable signal, <span style="color:#3333ff;">Blue</span>: Vdd1 with controlled ramp</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/soft-start-vdd1-vdd2.png" alt="Oscilloscope capture showing Vdd2 rising after Vdd1, confirming sequenced power-up">
  <figcaption>Power sequencing confirmed — <span style="color:#cc7700;">Orange</span>: Vdd1 rises first, <span style="color:#3333ff;">Blue</span>: Vdd2 follows with a delay, as designed</figcaption>
</figure>

**Load regulation** was tested by switching between 30 &Omega; (100 mA) and 15 &Omega; (200 mA) loads — a 100 mA step change. The output showed a brief transient and recovered within a few hundred microseconds, confirming that the Type 3 compensation was providing adequate loop bandwidth for load transients.

<figure>
  <img src="/Blog/assets/Images/Buck-Converter/load-regulation.png" alt="Oscilloscope capture of output voltage during a step load change from 30 Ohm to 15 Ohm, showing transient response and recovery">
  <figcaption>Load regulation — output voltage response during a step change from 30 &Omega; to 15 &Omega; (100 mA step), showing recovery within hundreds of microseconds</figcaption>
</figure>

## Process and Lessons Learned

This project spanned roughly seven months, including a pause in the middle. The iterative simulation-first approach paid off: by the time I ordered the PCB, the circuit was well-characterized and most of the debugging was about real-world deviations (reference drift, diode placement) rather than fundamental design flaws.

Key takeaways:

- **Compensation matters more than topology.** The jump from Type 1 to Type 3 compensation transformed a sluggish, marginally stable converter into one with fast transient response and comfortable phase margin — using the same power stage.
- **Design for your actual supply range.** The sawtooth generator failure at 4.8 V was a direct consequence of designing the comparator thresholds for exactly 5 V. Adding margin to the operating range would have prevented this.
- **Small hardware mistakes have outsized effects.** A single reversed diode completely defeated the soft-start circuit. Careful DRC and polarity marking are not optional.
- **Debug headers are essential.** Having probe points for Vsw, Vout, Vcontrol, and the sawtooth output made it possible to trace the root cause of every issue I encountered on the board.

<div style="margin-top:40px; padding-top:20px; border-top:1px solid #e5e7eb;">
  <p style="font-size:0.85rem; color:#6b7280; margin-bottom:12px; font-weight:600; text-transform:uppercase; letter-spacing:0.05em;">Skills & Technologies</p>
  <div style="display:flex; flex-wrap:wrap; gap:8px;">
    <span class="tag">Analog Circuit Design</span>
    <span class="tag">Power Electronics</span>
    <span class="tag">Control Theory</span>
    <span class="tag">Loop Compensation</span>
    <span class="tag">LTspice</span>
    <span class="tag">KiCad</span>
    <span class="tag">PCB Layout (4-layer)</span>
    <span class="tag">JLCPCB</span>
    <span class="tag">SMD Soldering</span>
    <span class="tag">Analog Discovery 3</span>
    <span class="tag">Hardware Debugging</span>
  </div>
</div>
