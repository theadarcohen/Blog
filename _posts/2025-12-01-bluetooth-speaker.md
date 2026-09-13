---
title: "Designing a Rechargeable Bluetooth Speaker from Scratch"
description: "A custom Bluetooth speaker — from power architecture and PCB design through 3D-printed enclosure to working product — built around an ESP32, Class D amplifier, and Li-ion battery on a 4-layer PCB."
date: 2025-12-01
thumbnail: "/assets/Images/Bluetooth-Speaker/final-assembly.png"
image: "/assets/Images/Social/bluetooth-speaker.jpg"
tags: [PCB Design, Bluetooth, ESP32, Power Electronics, 3D Printing, KiCad]
---

<div class="post-meta-bar" style="display:flex; flex-wrap:wrap; gap:16px; font-size:0.88rem; color:#6b7280; margin-bottom:32px; padding:16px 0; border-bottom:1px solid #e5e7eb;">
  <span><strong>Timeframe:</strong> December 2025</span>
  <span><strong>Role:</strong> Solo project</span>
  <span><strong>Type:</strong> Self-directed learning</span>
  <span><strong>Tools:</strong> KiCad, JLCPCB, Fusion 360, Cura, Arduino IDE, Creality Ender-3 S1 Pro</span>
  <span><strong>Read time:</strong> ~10 min</span>
</div>

## Introduction

I wanted to go through the full lifecycle of building an electronic product — not just a circuit on a breadboard, but a complete device with a custom PCB, rechargeable battery, wireless audio, and an enclosure you can hold in your hand. So I designed and built a portable Bluetooth speaker from scratch: schematic design, component selection, 4-layer PCB layout, firmware, and a 3D-printed case — all the way to a working product that streams music wirelessly.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/block-diagram.png" alt="Initial block diagram showing the five main modules: USB-C, charging, battery, Bluetooth, and amplifier with speaker">
  <figcaption>The initial block diagram — five modules: USB-C power input, battery charging, Li-ion battery, Bluetooth controller, and audio amplifier with speaker</figcaption>
</figure>

## Power Architecture

The power section was the most design-intensive part of the project, with several subsystems that needed to work together to safely charge, protect, and regulate a single-cell Li-ion battery.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/power-section-schematic.png" alt="Full schematic of the power section showing USB-C input, battery charger, battery management, power path selection, and LDO">
  <figcaption>The power section schematic — USB-C input, TP4056 charger, BQ29702 battery management, power path selection, and TLV75533 LDO</figcaption>
</figure>

### Power Budget

I estimated the system's power consumption across two operating modes. In **maximum consumption** — Bluetooth streaming with the amplifier active — the ESP32 draws ~100 mA and the amplifier draws ~235 mA (at 3.3 V supply, 8 &Omega; speaker, ~0.7 W output, ~90% efficiency), giving a total of roughly **335 mA at 3.3 V (~1.1 W)**. In **low-power mode** — when the battery is low or USB is connected — Bluetooth is disabled and the amplifier is shut down, dropping consumption to around **40 mA**.

### Battery and Charging

The battery is a Samsung INR18650-30Q Li-ion cell with 2950 mAh nominal capacity, 3.6 V nominal voltage, and a 2.5 V minimum discharge voltage. It connects to the board through a screw terminal.

The **TP4056** linear charger handles the CC/CV charging profile, with a 3 k&Omega; R<sub>PROG</sub> resistor setting the charge current to **400 mA**. This leaves 100 mA of headroom from the USB's 500 mA budget for the rest of the system, allowing the ESP32 to be programmed while the battery charges. Two indicator LEDs (via JST connectors) show charging status: one for "charging" and one for "charge complete."

### Battery Protection

The **BQ29702** battery management IC protects the cell from overcharge (>4.35 V), over-discharge (<2.8 V with hysteresis), and overcurrent. It controls two N-channel MOSFETs (IRF7341, dual NMOS in a single package) that disconnect the battery when a fault is detected. The overcurrent threshold, set by the MOSFETs' R<sub>DS(on)</sub> of ~50 m&Omega;, works out to approximately 1.6 A.

### Power Path Selection

When USB is connected, the system should run from USB power, not the battery. A **P-channel MOSFET** and a **Schottky diode** handle this automatically: the PMOSFET passes battery voltage when USB is absent, and the Schottky conducts USB power when USB is present, reverse-biasing the PMOSFET and disconnecting the battery. When USB is detected, the firmware also enters low-power mode and a hardware circuit disables the amplifier immediately — without waiting for the MCU to react — to keep current draw within the USB charging budget.

### Voltage Regulation

I initially considered a buck converter, but the small voltage difference between battery (~3.6 V) and output (3.3 V) and the moderate current draw made an **LDO** the better choice — simpler, quieter, and more stable. The **TLV75533** provides a fixed 3.3 V output with a maximum dropout of 238 mV at 500 mA, internal soft-start, and foldback current limiting. A slide switch on the LDO's enable pin serves as the system's power switch.

## Audio Section

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/audio-section-schematic.png" alt="Schematic of the audio section showing the MAX98357A Class D amplifier with I2S input and differential speaker output">
  <figcaption>The audio section — MAX98357A Class D amplifier receiving I2S audio and driving an 8 &Omega; speaker through differential outputs</figcaption>
</figure>

The audio amplifier is a **MAX98357A**, a Class D amplifier with a built-in DAC that accepts digital audio over an **I2S** interface. It supports sample rates from 8 kHz to 96 kHz at 16/24/32-bit resolution. I configured it for the left channel (SD_MODE pin pulled to an appropriate voltage via a 2.2 k&Omega; resistor) with a gain of **12 dB** (GAIN_SLOT pin tied to ground).

At 3.3 V supply and 8 &Omega; load, the amplifier delivers approximately **0.7 W** of output power at around **90% efficiency**. When the amplifier's SD_MODE pin is pulled low — either by the hardware USB-detect circuit or by the ESP32's GPIO — the amplifier enters shutdown mode, drawing sub-microamp current.

The speaker connects to the board through a screw terminal, keeping it off-board for flexibility.

## Bluetooth and MCU

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/bluetooth-section-schematic.png" alt="Schematic of the Bluetooth section showing the ESP32-WROOM-32E module with USB-to-UART, boot control, and I2S connections">
  <figcaption>The Bluetooth section — ESP32-WROOM-32E module with CP2102N USB-to-UART bridge, boot/reset buttons, and I2S output to the amplifier</figcaption>
</figure>

I initially planned to use a Silicon Labs **EFR32BG22** BLE chip, going as far as designing the full schematic with an RF matching network and PCB antenna. After completing the design, I realized that BLE doesn't support audio streaming — it requires Bluetooth Classic with the **A2DP** profile. This was a significant lesson in verifying protocol-level requirements before committing to a chip.

I switched to the **ESP32-WROOM-32E** module, which supports Bluetooth Classic A2DP natively and comes with a built-in PCB antenna, crystal, and flash — eliminating the need for an RF matching network, external crystal, and antenna design. The ESP32 streams audio over I2S to the MAX98357A amplifier using the open-source **BluetoothA2DPSink** library.

A **CP2102N** USB-to-UART bridge enables programming the ESP32 over the same USB-C connector used for charging. The boot sequence uses two buttons: holding the Boot button while pressing Reset puts the ESP32 into download mode for flashing firmware.

### Battery Monitoring

The ESP32 monitors both battery voltage and USB presence through its ADC. A resistor divider halves the battery voltage for the 12-bit ADC, and a GPIO-controlled MOSFET enables the divider only during measurement to avoid constant current drain. When the battery drops below 3.5 V, the firmware disables Bluetooth, shuts down the amplifier, lights a low-battery LED, and enters light sleep — waking every 500 ms to recheck. A 50 mV hysteresis band (3.5 V/3.55 V) prevents oscillation near the threshold.

## PCB Design

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/jlcpcb-stackup.png" alt="JLCPCB JLC04161H-7628 4-layer PCB stackup showing layer thicknesses and dielectric constants">
  <figcaption>The JLC04161H-7628 stackup — a 4-layer board with 1 oz outer copper and 0.5 oz inner copper</figcaption>
</figure>

The PCB is a **4-layer** design using the JLC04161H-7628 stackup from JLCPCB: signal layers on top and bottom, with two internal ground planes. Signal traces are 0.3 mm for general routing and 0.5 mm for power, with minimum spacing of 0.63 mm (three times the dielectric thickness). The board measures **56 &times; 47 mm**, not including the ESP32's PCB antenna overhang.

I created custom footprints for several components, including the TP4056, MAX98357A (WLP/BGA package), IRF7341, CP2102N, USB-C connector, and various connectors.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/component-placement.png" alt="Component placement view of the PCB showing organized module groupings">
  <figcaption>Component placement — modules grouped by function following datasheet layout recommendations</figcaption>
</figure>

I ordered the boards from **JLCPCB** with full assembly (5 boards, all SMD components placed). The total cost was about $166 plus $28 shipping, with a significant portion of the assembly cost coming from the MAX98357A's tiny BGA package. The boards arrived about three weeks after ordering.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/received-pcb.png" alt="Photos of the assembled PCB as received from JLCPCB">
  <figcaption>The assembled PCBs as received from JLCPCB</figcaption>
</figure>

## Testing and Bring-Up

Testing proceeded incrementally, verifying each subsystem before moving on.

**ESP32 communication** — With just the LDO enabled (EN pin jumpered to VDD), I confirmed UART communication through the CP2102N and ran Bluetooth Serial examples successfully.

**LEDs and switch** — Connected the indicator LEDs and power switch via JST cables, verified the switch toggles the LDO, and confirmed correct LED behavior. I discovered that GPIO14 (low-battery indicator) starts at an intermediate voltage (~2.2 V) after reset, lighting the LED unintentionally — resolved in firmware by explicitly driving the pin low at startup.

**Battery operation** — Charged the Samsung 18650 cell from 3.54 V to 4.24 V, monitoring voltage over time. The TP4056 charged at the expected 400 mA (1 V measured on R<sub>PROG</sub>) and correctly transitioned from CC to CV mode, with the standby LED lighting when charging completed.

**Audio streaming** — Using the BluetoothA2DPSink library, the ESP32 appeared as "MyMusic" on Bluetooth, paired successfully, and played audio through the speaker. I encountered an issue where USB voltage lingered on the VBUS line after disconnection (due to the CP2102N errata causing D+ to float high, forward-biasing the ESD protection diode to VBUS), which triggered the amplifier shutdown circuit. A power-cycle of the LDO switch clears this condition.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/board-with-battery-speaker.png" alt="The PCB with battery and speaker connected during audio testing">
  <figcaption>Audio testing — the board with the 18650 battery and 8 &Omega; speaker connected, streaming music over Bluetooth</figcaption>
</figure>

## Firmware

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/code-structure.png" alt="Flowchart showing the firmware state machine with battery monitoring, USB detection, and power mode transitions">
  <figcaption>The firmware state machine — monitors battery voltage and USB status, transitioning between normal operation, Bluetooth-only inhibit, and full low-power sleep</figcaption>
</figure>

The firmware runs on the Arduino framework at **80 MHz** (reduced from 240 MHz for lower power consumption). A dedicated FreeRTOS task on Core 1 monitors the system state every 200 ms, implementing three operating modes:

- **Normal mode** — Bluetooth A2DP streaming active, amplifier enabled, low-battery LED off.
- **USB connected** — Bluetooth disabled to maximize charging current; amplifier remains available (hardware shutdown handles it if needed).
- **Low battery** — Bluetooth disabled, amplifier shut down via GPIO, low-battery LED lit, ESP32 enters light sleep waking every 500 ms to recheck voltage. GPIO states are held during sleep using `gpio_hold_en()`.

The ADC readings are calibrated using `esp_adc_cal` and doubled to account for the resistor divider. The battery threshold uses hysteresis (3500 mV low / 3550 mV high) to prevent oscillation.

## 3D-Printed Enclosure

The enclosure was designed in **Fusion 360** and printed on a **Creality Ender-3 S1 Pro** with PLA filament, sliced through **UltiMaker Cura**. I exported the PCB's 3D model from KiCad into Fusion 360 to ensure accurate fitouts.

The design went through several iterations:

1. **Test box** — A simple snap-fit box to test a clip mechanism. The PLA was less flexible than expected; the snap-fit was too tight to assemble. Lesson: PLA needs larger tolerances for snap fits.
2. **Board enclosure** — A partial case for just the PCB, with cutouts for the USB-C port and antenna. Good fit, but I realized I should have added mounting holes to the PCB to secure it.
3. **Battery holder** — A custom cradle for the 18650 cell, designed to accept metal contacts salvaged from a commercial battery holder. Dimensions calculated from the battery spec (65 mm + spring compression + tolerances = 79.1 mm total length).
4. **Clip test** — A separate test piece to validate a leg-through-hole clamping mechanism for attaching the lid. The legs (2 &times; 2 mm) fit through holes (2.5 &times; 2.5 mm) with alignment features. Tight but functional.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/enclosure-model.png" alt="3D model of the combined enclosure housing the PCB, battery, and speaker with ventilation holes for sound">
  <figcaption>The combined enclosure model — housing the PCB, 18650 battery, and speaker, with ventilation holes for audio output</figcaption>
</figure>

The final enclosure integrates all three compartments: PCB, battery, and speaker (with sound holes). The lid holds the LEDs and slide switch, glued in place with super glue. Total print time was about 4.5 hours for the body and 1 hour 40 minutes for the lid.

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/final-assembly.png" alt="Photo of the PCB, battery, and speaker next to the 3D-printed enclosure">
  <figcaption>The final assembly — PCB, 18650 battery, and speaker alongside the 3D-printed enclosure</figcaption>
</figure>

<figure>
  <img src="/Blog/assets/Images/Bluetooth-Speaker/charging-status.png" alt="Photos showing the speaker charging (left) with blue LED and charge complete (right) with green LED">
  <figcaption>Charging in progress (left, blue LED) and charge complete (right, green LED)</figcaption>
</figure>

## Lessons Learned

- **Verify protocol-level requirements before choosing a chip.** I designed an entire BLE subsystem around the EFR32BG22 before discovering that BLE doesn't support audio streaming — only Bluetooth Classic A2DP does. This cost significant design time but saved me from ordering an unusable board.
- **End-to-end product design is fundamentally different from circuit design.** Beyond the schematic, there are thermal pads, JST connector wire lengths, LED placement through an enclosure, button accessibility, PLA flexibility tolerances, and dozens of other integration details that only surface when you try to build a complete product.
- **Power architecture decisions cascade through the entire design.** The choice to use an LDO instead of a buck converter, the 400 mA charge current limit to leave headroom for programming, the hardware amplifier shutdown to avoid USB current spikes — each of these required thinking about the system as a whole, not just individual subsystems.
- **3D printing iteration is fast but PLA has real limitations.** Snap-fit mechanisms that work in ABS or nylon don't work in rigid PLA. The printer resolves features down to about 0.5 mm, which is fine for enclosures but limits the complexity of mechanical interlocks.

<div style="margin-top:40px; padding-top:20px; border-top:1px solid #e5e7eb;">
  <p style="font-size:0.85rem; color:#6b7280; margin-bottom:12px; font-weight:600; text-transform:uppercase; letter-spacing:0.05em;">Skills & Technologies</p>
  <div style="display:flex; flex-wrap:wrap; gap:8px;">
    <span class="tag">KiCad</span>
    <span class="tag">PCB Design (4-Layer)</span>
    <span class="tag">Power Electronics</span>
    <span class="tag">Li-ion Battery Management</span>
    <span class="tag">Bluetooth Classic (A2DP)</span>
    <span class="tag">ESP32</span>
    <span class="tag">I2S Protocol</span>
    <span class="tag">Class D Amplifier</span>
    <span class="tag">USB-C</span>
    <span class="tag">3D Printing (Fusion 360 / Cura)</span>
    <span class="tag">Arduino / C++</span>
    <span class="tag">JLCPCB Assembly</span>
    <span class="tag">Firmware Development</span>
    <span class="tag">Product Design</span>
  </div>
</div>
