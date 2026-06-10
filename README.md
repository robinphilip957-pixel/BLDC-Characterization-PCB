# Compact Sensorless BLDC Motor Characterization PCB

<p align="center">
  <img src="Docs/Hardware Setup.png" alt="Hardware Setup" height="300"/> &nbsp;&nbsp;&nbsp;
  <img src="Docs/Patent Status.png" alt="Patent Status" height="300"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Patent-Published-green"/>
  <img src="https://img.shields.io/badge/Application%20No-202641026987-blue"/>
  <img src="https://img.shields.io/badge/License-All%20Rights%20Reserved-red"/>
</p>

A compact single PCB-based sensorless system for analyzing the performance of Brushless DC (BLDC) motors, fully designed and developed in EasyEDA Pro. The system integrates an Electronic Speed Controller (ESC), a current and voltage sensing unit, and an ESP32 microcontroller on a single printed circuit board — eliminating the need for mechanical sensors and external test equipment.

The ESP32 generates PWM signals to drive the BLDC motor through the ESC and incrementally varies the PWM duty cycle across multiple speed levels. Real-time motor input current and voltage are measured by the sensing unit and processed by the microcontroller, along with motor constants and calibration data, to estimate motor operating parameters including rotational speed, torque, and electrical power. The system generates a comprehensive motor performance profile comprising relationships among speed, torque, power consumption, and efficiency — providing a compact, low-cost, and reliable platform for BLDC motor performance characterization.

> **Patent Application No:** 202641026987
> **Status:** Published
> **Title:** COMPACT SINGLE PCB-BASED SENSOR-LESS SYSTEM AND METHOD FOR ANALYZING PERFORMANCE OF BRUSHLESS DC (BLDC) MOTOR



## Component ICs Used
- ESP32 (Microcontroller — PWM Generation, ADC Sensing, Performance Profiling)
- LMR51420YDDCR × 3 (Synchronous Buck Converters — BAT+ → 3.3V / 5V / 12V)
- Current & Voltage Sensing Module (Motor Input Current and Bus Voltage Measurement)
- XT60U-M (Main LiPo Battery Input Connector)
- XT60-F × 2 (ESC Power Output and Motor Connector)
- PM254V-11-16-H85 × 2 (16-Pin Headers — ESP32 and ESC Interface)
- PM254V-11-09-H85 (9-Pin Header — Current/Voltage Sensor Module Interface)
- HX PH254-01-03-Z-L11.5 (3-Pin Header — Signal/Phase Connector)
- White LED + R7 1kΩ (Power Indicator)



## Power Architecture

| Rail  | Source    | Buck Converter     | Inductor | Feedback Divider (Rtop / Rbot) | Powered Subsystem           |
|:-----:|:---------:|:------------------:|:--------:|:------------------------------:|:---------------------------:|
| 3.3V  | BAT+      | LMR51420YDDCR (U1) | 3.3µH    | 100kΩ / 22.1kΩ                 | ESP32, Logic                |
| 5V    | BAT+      | LMR51420YDDCR (U2) | 4.7µH    | 100kΩ / 13.7kΩ                 | Sensing Module, Peripherals |
| 12V   | BAT+      | LMR51420YDDCR (U3) | 8.2µH    | 100kΩ / 5.23kΩ                 | ESC Supply                  |
| BAT+  | Li-ion 4S | XT60U-M (CN1)      | —        | —                              | Main Power Input            |



## Design Notes

### Buck Converter Feedback Dividers (100kΩ / 22.1kΩ, 13.7kΩ, 5.23kΩ)
The LMR51420 regulates its FB pin to 0.6V. The top and bottom resistors of the feedback divider set the output voltage via the relation `Vout = 0.6 × (1 + Rtop/Rbot)`. A 100kΩ top resistor keeps the divider current in the microamp range (~27µA for 3.3V rail), avoiding unnecessary battery drain while keeping the node low-impedance enough to resist noise.

### EN/UVLO Dividers (274kΩ / 27kΩ) — One Pair Per Converter
Each buck converter has a dedicated 274kΩ/27kΩ resistor pair on its EN pin. The chip enables at ~1.23V and disables at ~1.08V on EN. Scaling with this divider sets the turn-on threshold to ~13.7V and turn-off to ~12.0V on the battery rail — a safe operating window for a 4S LiPo pack. This hysteresis prevents the regulators from chattering when the battery voltage is near the threshold. Divider current (~56µA at 16.8V) is negligible.

### Inductor Selection (3.3µH, 4.7µH, 8.2µH)
Inductor values are chosen from the LMR51420 datasheet's recommended table for its 1.1MHz switching frequency. Larger inductance reduces ripple current but slows transient response and increases physical size. These values strike the right compromise for a 2A output — enough ripple for stable operation without overstressing the inductor or output capacitors.

### Input Capacitor Stack (0.1µF + 2.2µF + 2×10µF per converter)
Three capacitor tiers handle noise across the full frequency spectrum:
- **0.1µF ceramic** — placed directly at the IC pins; kills fast high-frequency switching spikes.
- **2.2µF ceramic** — handles mid-frequency noise and provides better real capacitance under DC bias than a single small part.
- **2×10µF ceramic** — bulk decoupling that supplies short bursts of current and smooths lower-frequency ripple. Multiple parts in parallel lower ESR and offset capacitance loss due to DC bias derating.

### Output Capacitors (2×10µF for 3.3V & 5V, 3×10µF for 12V)
The LMR51420 targets ~20µF effective output capacitance at 1.1MHz for stability. Ceramic capacitors lose rated value under DC bias, so two 10µF parts in parallel are used for the 3.3V and 5V rails. The 12V rail uses three 10µF parts because DC bias derating is more severe at higher voltages — three parts ensure the effective capacitance stays near the required value.

### Bootstrap Capacitor (0.1µF)
Required by the LMR51420 datasheet to power the internal high-side MOSFET gate driver. This is a fixed specification value, not a tuning choice.

### ADC Input Filter (1kΩ series + 100nF)
A first-order RC low-pass filter on each ADC line protects the ESP32 from high-frequency switching noise coupled from the ESC and power rails. With R = 1kΩ and C = 100nF, the cutoff frequency is `fc = 1/(2π × 1kΩ × 100nF) ≈ 1.6kHz` — fast enough for telemetry sampling while attenuating switching noise in the tens-to-hundreds of kHz range. A much larger resistor would destabilise ADC readings; a much smaller one would let too much noise through.

### Voltage Divider Filter (10nF)
A 10nF capacitor placed across the lower resistor of the voltage sensing divider smooths the divider node, preventing the ESP32 ADC from sampling transient spikes caused by switching events elsewhere on the board. With the divider's Thevenin resistance, the cutoff lands in the low-kilohertz range — fast enough for accurate readings, slow enough to filter switching artifacts.

### PWM Series Resistor (47Ω at ESP32 side)
Placed in series with the PWM output line close to the ESP32, this resistor damps fast signal edges and prevents ringing/reflections on the cable between the MCU and ESC. At 47Ω it slows the edge just enough to reduce EMI and protect the MCU pin without distorting the pulse timing or confusing the ESC. An optional 22–33Ω resistor or ferrite bead at the ESC end can further absorb residual high-frequency energy if needed.



## Sensing Architecture

**Current and Voltage Sensing Module (U7 — 9-pin header):**

The sensing module measures real-time motor input current and bus voltage. Its analog outputs are fed directly into the ESP32 ADC through the RC input filters described above.

| **Signal** | **Net**  | **Description**                   |
|:----------:|:--------:|:---------------------------------:|
| I_OUT      | I_OUT    | Analog current output → ESP32 ADC |
| V_OUT      | V_OUT    | Analog voltage output → ESP32 ADC |
| Power      | 5V / 12V | Module supply rails               |



## ESP32 Pinout

| **Function**            | **GPIO** |
|:-----------------------:|:--------:|
| PWM Output (ESC Signal) | IO18     |
| Current ADC (I_OUT)     | IO34     |
| Voltage ADC (V_OUT)     | IO35     |



## How It Works

1. **PWM Sweep** — ESP32 starts at a minimum duty cycle and incrementally increases PWM to the ESC in defined steps, operating the BLDC motor at multiple speed levels.
2. **Sensing** — At each step, the current and voltage sensing module measures real-time motor input current and bus voltage. Analog outputs are filtered and read by the ESP32 ADC.
3. **Estimation** — The microcontroller processes ADC readings along with motor constants (KV rating, winding resistance) and calibration data to estimate RPM, torque, and electrical power.
4. **Profiling** — All data points are compiled to generate a comprehensive performance profile: Speed vs Torque, Speed vs Power, Speed vs Efficiency.



## References
- [ESP32 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf)
- [LMR51420 Buck Converter Datasheet](https://www.ti.com/lit/ds/symlink/lmr51420.pdf)
- [XT60 Connector Datasheet](https://www.amass.com/en/index.php?a=proinfo&id=31)
- [ESP32 ADC Reference](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc.html)
- [GreatScott YouTube](https://www.youtube.com/@greatscottlab)
- [Robert Feranec YouTube](https://www.youtube.com/@RobertFeranec_)
- [Phill's Lab YouTube](https://www.youtube.com/watch?v=_Hfzq1QES-Q)

---

> ⚠️ **Intellectual Property Notice**
> This design is protected under Indian Patent Application No. **202641026987** (Published).
> All hardware designs, schematics, and associated files are the intellectual property of Robin Philip.
> Reproduction, modification, or commercial use without explicit written permission is prohibited.

Author: Robin Philip