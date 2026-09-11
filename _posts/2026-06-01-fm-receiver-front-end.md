---
title: "Designing an FM Receiver Front End from Discrete Transistors"
description: "A design study of a superheterodyne FM receiver built from discrete transistors — the 98 MHz low-noise amplifier and a tunable local oscillator are built and measured, while the rest of the radio was never integrated."
date: 2026-06-01
thumbnail: "/assets/Images/FM-Receiver/lna-build-hero.png"
tags: [RF Design, Analog Design, Impedance Matching, LTspice, NanoVNA]
math: true
---

<div class="post-meta-bar" style="display:flex; flex-wrap:wrap; gap:16px; font-size:0.88rem; color:#6b7280; margin-bottom:32px; padding:16px 0; border-bottom:1px solid #e5e7eb;">
  <span><strong>Timeframe:</strong> February – June 2026</span>
  <span><strong>Role:</strong> Solo project</span>
  <span><strong>Type:</strong> Self-directed learning</span>
  <span><strong>Status:</strong> Design study (on hold)</span>
  <span><strong>Tools:</strong> LTspice, KiCad, Analog Discovery 3, NanoVNA, TinySA</span>
  <span><strong>Read time:</strong> ~9 min</span>
</div>

## Introduction

I wanted to learn RF design the hard way: by building an FM radio from discrete transistors, with no receiver ICs to hide the interesting parts. Every block would be designed from first principles, simulated in LTspice, built by hand, and measured against its prediction.

The full radio never came together. I built each kind of circuit at audio and 1 MHz frequencies first, then moved to the FM band, where the low-noise amplifier and a tunable local oscillator now work at around 100 MHz. The 98 MHz mixer, the IF stage, and the demodulator stayed on paper — the project grew far larger than I had planned, and I set it aside to move on. This page is a design study of how far it got. The highlights:

- **An LNA that measured 8.6 dB instead of the predicted 18 dB** — and a root-cause analysis that traced the gap to the transistor's own 9.2 dB ceiling.
- **A tunable oscillator covering 87–116.8 MHz**, built around a hand-wound single-turn coil after two failed attempts.
- **Hand-wound inductors, matching networks, and filters at 98 MHz** that landed within a few percent of their calculated values.

## The Architecture

The receiver follows the classic **superheterodyne** architecture. The FM broadcast band in Israel spans 87.5–108 MHz, with stations spaced 200 kHz apart. Filtering one station out of that band directly at 100 MHz would demand an impractically sharp filter, so a mixer and **local oscillator (LO)** first shift the chosen station down to a fixed **intermediate frequency (IF)** of 10.7 MHz, where selective filters and high gain are far easier to build.

I placed the LO above the station frequency, tuning from 98.2 to 118.7 MHz. That puts the **image frequency** — the second input that a mixer maps onto the same IF — at 108.9 MHz or higher, just outside the FM band, so the front-end filter can reject it.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/receiver-block-diagram.svg" alt="Block diagram of the superheterodyne FM receiver: antenna, input match, LNA, RF band-pass and output match, mixer with local oscillator, then channel filter, IF amplifier, demodulator, audio amplifier and speaker, each colored by build status">
  <figcaption>The planned receiver, colored by how far each block got — the RF front end and the audio stage exist in hardware; the IF chain does not</figcaption>
</figure>

## Groundwork at 1 MHz

First, I built one of each kind of block at low frequencies, where a breadboard and an Analog Discovery 3 suffice and parasitics are forgiving.

The **audio amplifier** — the last block in the chain — pairs a common-emitter stage with a gain of about 9 and a class-AB push-pull output built from Darlington pairs; simpler emitter followers clipped after just 20–160 mV into the 8 &Omega; speaker. With an electret microphone at the input, speech reached the speaker with a gain of about 5. That's less than the stages promised separately: the Darlington input loaded the first stage more than the constant-β SPICE model predicted.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/audio-amp-breadboard.png" alt="Breadboard with an electret microphone, a common-emitter stage and a class-AB Darlington output stage wired to an 8 ohm speaker">
  <figcaption>The two-stage audio amplifier on a breadboard — electret microphone in, 8 &Omega; speaker out</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/audio-amp-speech.png" alt="Oscilloscope capture of microphone voltage (orange) and speaker voltage (blue) while speaking into the microphone, showing the inverted and amplified speech waveform">
  <figcaption>Speaking into the microphone: input (orange) and speaker voltage (blue), amplified and inverted by the common-emitter stage</figcaption>
</figure>

I also built three **Colpitts oscillators** around 1 MHz. The tunable one came in about 12% low — until I added the oscilloscope probe's 24 pF and the transistor's measured 38 pF base-emitter capacitance to the tank, and the recalculated 859–923 kHz matched the measured 865–925 kHz.

Of the three **mixers** I built, the passive diode ring was the most instructive: toroid transformers I wound myself and four Schottky diodes matched to a 0.199 V forward drop. At first the 4.7 k&Omega; drive resistors limited the diode current to about 234 &micro;A and the diodes barely switched. With 1 k&Omega;, the difference product reached 0.64 of the RF amplitude — the $$2/\pi$$ that ideal switching predicts. The sum product, which should be equally strong, stayed several times weaker, pointing to an asymmetry (the switching duty cycle measured 43%) that I never fully tracked down.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/diode-ring-mixer.png" alt="Breadboard with a Colpitts oscillator, two hand-wound toroid transformers, four diodes and a common-emitter RF amplifier, above the matching LTspice schematic">
  <figcaption>The diode-ring mixer with its oscillator and RF amplifier, above the LTspice schematic (shown with the original 4.7 k&Omega; resistors)</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/diode-ring-mixer-spectrum.png" alt="Spectrum showing the 1 MHz RF input at about 100 millivolts and the mixer's difference product just under 400 kilohertz at about 62 millivolts, with a smaller sum product near 2.4 megahertz">
  <figcaption>After the fix: the 1 MHz RF input (blue) and the mixer output (orange), with the difference product at roughly 0.64 of the RF amplitude</figcaption>
</figure>

## Designing at 100 MHz

At 98 MHz, parasitics stop being small corrections and become circuit elements. On a NanoVNA, the 2N2222A transistor showed about 41 pF between base and emitter and 14 pF between base and collector, a 2.2 k&Omega; resistor looked like 2 k&Omega; in parallel with 1 pF, and 10 pF capacitors read 10–20% high. From here on, every critical value in my designs came from a measurement rather than a label.

For inductors of tens to hundreds of nanohenries, I wound air-core coils on 10 mm forms — first a pen, then 3D-printed PLA cylinders — measured each one, and trimmed the inductance by spreading the turns apart. With characterized parts, I designed **L-section matching networks**, which transform a resistance $$R_s$$ up to $$R_p$$ at one frequency with a single inductor and capacitor. The ratio sets the network's quality factor, and with it the bandwidth:

$$Q = \sqrt{\frac{R_p}{R_s} - 1}, \qquad BW \approx \frac{f_0}{Q}$$

A 47 &Omega; to 510 &Omega; network measured 515 &Omega; at 95.5 MHz against a calculated 510 &Omega; at 97.4 MHz, and a parallel RLC band-pass filter measured 100 MHz, 1.66 k&Omega; and 8.6 MHz of bandwidth against a predicted 104 MHz, 1.5 k&Omega; and 9 MHz. The exception was a 50 &Omega; to 5 &Omega; match that resonated at 87 MHz instead of 99 MHz, most likely from a larger coil loop than I had measured and solder heat raising the capacitance.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/rf-bandpass-filter-vna.png" alt="NanoVNA screen showing the impedance magnitude of the band-pass filter peaking at 1.66 kilohms at 100 megahertz across the 87.5 to 108 megahertz sweep">
  <figcaption>The RF band-pass filter on the NanoVNA — a 1.66 k&Omega; peak at 100 MHz across the FM band</figcaption>
</figure>

## The Low-Noise Amplifier

I chose a **common-base** amplifier for the first stage: it isolates the output from the input far better than a common-emitter stage, and an L-match raises its few-ohm input impedance to the antenna's assumed 50 &Omega;. A tuned circuit at the collector sets the gain and doubles as the RF band-pass filter, and a second L-match brings the output to 50 &Omega;, its inductor merged with the tank's into one hand-wound coil. With the component values I actually had, the calculation predicted **18 dB of gain and 26 MHz of bandwidth**.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/lna-build-schematic.png" alt="The common-base LNA on perfboard, with a red three-turn coil on a white 3D-printed form and a single-loop input coil, next to its KiCad schematic showing a 2N2222A with antenna and load matching networks">
  <figcaption>The LNA on perfboard next to its schematic — the three-turn coil on a 3D-printed form combines the tank and output-match inductors (L2 &#124;&#124; L5), and the single loop at the bottom is the input-match inductor (L1)</figcaption>
</figure>

I measured the LNA on the NanoVNA with 20 dB attenuators on both ports, to protect the analyzer and keep the amplifier out of saturation. After spreading the coil turns until the response centered on 98 MHz, the gain was **8.6 dB**.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/lna-s-parameters.png" alt="NanoVNA screen showing the LNA forward gain S21 of 8.61 dB and input reflection S11 of minus 8.68 dB at 97.955 megahertz, with the input impedance on a Smith chart">
  <figcaption>LNA S-parameters at 98 MHz: 8.6 dB of forward gain (S21, cyan) and &minus;8.7 dB input reflection (S11, yellow)</figcaption>
</figure>

I worked through the likely causes one at a time:

- **Saturation?** Raising the attenuation to 30 dB left the gain unchanged.
- **Component parasitics?** The resistor and capacitor errors I had measured at 98 MHz explained a small loss, not 10 dB.
- **Mismatch?** Input reflection of &minus;8.7 dB, output reflection of &minus;19.4 dB, and reverse gain of &minus;16 dB put the mismatch loss at around 1 dB at most.

That left the transistor. What limits power gain is $$f_{max}$$, the frequency where a perfectly matched transistor's power gain falls to 0 dB. Using the datasheet's worst-case transition frequency and feedback time constant:

$$f_{max} = \sqrt{\frac{f_T}{8\pi\, r_{bb'} C_{b'c}}} \approx 282\ \text{MHz}$$

Maximum available gain falls at about 20 dB per decade below $$f_{max}$$, so at 98 MHz:

$$\text{MAG} \approx 20 \log_{10}\frac{f_{max}}{f} = 20 \log_{10}\frac{282}{98} \approx 9.2\ \text{dB}$$

That figure assumes 20 mA of collector current; at my 5.5 mA bias the ceiling is lower still. **The measured 8.6 dB wasn't a design error — it was the 2N2222A's limit**, and no matching network could have fixed it. I redesigned the LNA around an **S9018**, an SMD transistor with a typical transition frequency of 1.1 GHz and about 1 pF of feedback capacitance, on a larger board with surface-mount passives. That version hasn't been measured.

## The Local Oscillator

The LO is a common-base Colpitts oscillator, tuned by a screwdriver-adjusted trimmer capacitor across the tank:

$$f_0 = \frac{1}{2\pi\sqrt{L\,(C_T + C_t)}}, \qquad C_T = \frac{C_1 C_2}{C_1 + C_2}$$

From the trimmer's measured capacitance range, plus the transistor's own base-collector capacitance, I solved for the coil and fixed capacitors.

The first two builds failed. The first oscillated only after I removed the trimmer, whose metal body I suspect was disturbing the coil's field, and the TinySA spectrum analyzer's 50 &Omega; input loaded the collector enough to stop oscillation outright. The second, sampled through a resistive divider, oscillated only at certain trimmer positions and wouldn't tune — most likely because the loop gain varied with the trimmer setting.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/local-oscillator.png" alt="Close-up of the local oscillator on perfboard with a single-turn coil and trimmer capacitor, next to its KiCad schematic showing a 2N2222A common-base Colpitts oscillator with a load matching network">
  <figcaption>The working local oscillator next to its schematic — the tank coil and output-match coil are combined into one hand-wound turn</figcaption>
</figure>

The third build tuned across 73.9–104.4 MHz, which exposed a planning mistake of my own: I had designed for 80–110 MHz before remembering that a high-side LO has to cover 98.2–118.7 MHz. I raised one tank capacitor from 11 pF to a measured 26 pF and replaced the coil with a single wider turn of about 50 nH, trimming its length until the range moved up. The final oscillator tunes **87–116.8 MHz** at about &minus;30 dBm, enough for stations up to about 106 MHz. Its output fell around 10 dB in the process, consistent with the loop gain roughly halving when that capacitor doubled.

<figure>
  <img src="/Blog/assets/Images/FM-Receiver/lo-tuning-range.png" alt="Two TinySA spectrum analyzer screens showing the oscillator output at 87.14 megahertz and minus 28.9 dBm on the left and 116.84 megahertz and minus 30.5 dBm on the right">
  <figcaption>The two ends of the tuning range on the TinySA: 87.1 MHz (left) and 116.8 MHz (right)</figcaption>
</figure>

## Where the Design Stands

The project is currently on hold. The rest of the chain is designed, though, so the roadmap is concrete:

1. **Move the LNA and the LO to the S9018 and measure them** — a direct test of the $$f_{max}$$ diagnosis, and more loop-gain headroom for the oscillator's output.
2. **Build the 98 MHz mixer**, designed as a BJT mixer with a Darlington input and a tuned 10.7 MHz load. My hand-wound coils only reach a quality factor of about 36 at 10.7 MHz, roughly 540 kHz of bandwidth, so a 180 kHz ceramic filter would set the channel selectivity.
3. **Build the IF amplifier and demodulator.** The quadrature detector multiplies the IF signal by a phase-shifted copy of itself, turning frequency deviation into voltage; in LTspice it recovers a 1 kHz tone from a modulated 10.7 MHz carrier.
4. **Connect the chain to the audio amplifier**, which already drives a speaker.

## Lessons Learned

- **Check the device's limits before the circuit's.** The datasheet's transition frequency and feedback time constant predicted the LNA's ceiling; I only looked after building it.
- **At 100 MHz, measure everything.** Resistors have capacitance, capacitors drift after soldering, and a coil's inductance depends on the exact form it's wound on. Designing with measured values is what made the matching networks land close. Next time I'd also build on prototype boards with a ground plane — on bare perfboard, every wire adds inductance.
- **Instruments are part of the circuit.** A 24 pF probe shifted an oscillator by 8%, and a 50 &Omega; analyzer input stopped one entirely.
- **Build it slow, and scope it small.** The 1 MHz versions taught me oscillator start-up and mixer switching cheaply. But designing every block of a full receiver took more time than I could sustain, and the RF blocks were never connected to each other — a narrower first goal would have reached a working signal path sooner.

<div style="margin-top:40px; padding-top:20px; border-top:1px solid #e5e7eb;">
  <p style="font-size:0.85rem; color:#6b7280; margin-bottom:12px; font-weight:600; text-transform:uppercase; letter-spacing:0.05em;">Skills & Technologies</p>
  <div style="display:flex; flex-wrap:wrap; gap:8px;">
    <span class="tag">RF Design</span>
    <span class="tag">Superheterodyne Architecture</span>
    <span class="tag">LNA Design</span>
    <span class="tag">Colpitts Oscillators</span>
    <span class="tag">Mixers</span>
    <span class="tag">Impedance Matching</span>
    <span class="tag">S-Parameters</span>
    <span class="tag">Discrete BJT Circuits</span>
    <span class="tag">Hand-Wound Inductors</span>
    <span class="tag">LTspice</span>
    <span class="tag">KiCad (Schematics)</span>
    <span class="tag">Analog Discovery 3</span>
    <span class="tag">NanoVNA</span>
    <span class="tag">TinySA</span>
    <span class="tag">3D Printing</span>
  </div>
</div>
