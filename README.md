# Universal 24 Bit ADC Board for Instrumentation Systems
Designed a universal 24 bit ADC board in EasyEDA around the TI ADS1256. It has 4 differential input channels with RC anti alias filtering, a 7.68 MHz crystal clock, a filtered external 2.5V reference, separate 5V analog and 3.3V digital supplies from my precision power board, and an SPI interface for ESP32, all on a 4 layer PCB.

**Author:** Devansh Sharma
**Version:** Board V1.1.1, Hardware V1.0
**Status:** Design complete (schematic, placement and routing). Physical bring up pending.

> **Note:** This README is a temporary placeholder. It is an AI generated overview of the completed design, written to give a quick idea of the project while the full documentation is being prepared. A detailed write up covering design decisions, layout, bring up and test results will replace it once the board has been assembled and tested.

## Overview

Most instrumentation projects need the same front end: a way to turn small analog voltages into accurate digital readings. Instead of building a new ADC stage for every project, this board provides one universal 24 bit acquisition module that any future sensor or measurement system can plug into.

The board contains only the converter and its support circuitry. Power and the precision reference come from the Universal Precision Power Board, and control and data transfer happen over SPI, typically with an ESP32.

**Provides:**

1. 24 bit delta sigma conversion using the ADS1256
2. 4 fully differential input channels (8 single ended inputs internally)
3. RC anti alias filtering on every input
4. Separate analog (5V) and digital (3.3V) supplies
5. External 2.5V precision reference input with local filtering
6. SPI interface through both a JST connector and a 2.54mm pin header
7. Onboard reset button

## System Architecture

```
Universal Precision Power Board
   ├→ 5V_AFE      → CN1 → AVDD (analog supply)
   ├→ 3V3_CLEAN   → CN2 → DVDD (digital supply)
   └→ ADC_REF_OUT → CN3 → RC filter → VREFP

CH1 to CH4 (CN4 to CN7) → RC filter → AIN0 to AIN7 → ADS1256 → SPI → ESP32 (U3 / J1)
                                                        ↑
                                              7.68MHz crystal (X2)
```

## ADC Core (ADS1256)

The **ADS1256IDBR** is a 24 bit, 8 channel delta sigma ADC with an onboard programmable gain amplifier (PGA gain 1 to 64), input buffer and digital filter. It supports data rates up to 30kSPS, which makes it suitable for strain gauges, thermocouples, photodetectors, bridge sensors and other slow, high precision signals.

The "24 bit" figure is the theoretical output word size. The effective noise free resolution depends on the data rate, gain and the quality of the supply and reference, which is why this board is paired with a dedicated low noise power board.

## Analog Inputs

The 8 analog inputs are grouped into **4 differential channels**:

<table>
<tr><th>Channel</th><th>Connector</th><th>ADC Inputs</th></tr>
<tr><td>CH1</td><td>CN4</td><td>AIN0 (+), AIN1 (−)</td></tr>
<tr><td>CH2</td><td>CN5</td><td>AIN2 (+), AIN3 (−)</td></tr>
<tr><td>CH3</td><td>CN6</td><td>AIN4 (+), AIN5 (−)</td></tr>
<tr><td>CH4</td><td>CN7</td><td>AIN6 (+), AIN7 (−)</td></tr>
</table>

Each channel has **301Ω series resistors** on both lines along with **100nF** and **100pF** capacitors. Together they form an RC anti alias filter that limits high frequency noise before it reaches the ADC and protects the inputs from fast transients. Each input connector also carries GND for cable shielding or sensor ground.

## Reference

The 2.5V precision reference is generated on the power board and brought in through CN3 as ADC_REF_OUT. On this board it passes through a **49.9Ω** resistor into VREFP, with **47µF + 100nF + 100pF** filtering right at the pin. VREFN returns to ground through a matching 49.9Ω resistor.

Any noise on the reference appears directly in every conversion, so this local RC filter is one of the most important parts of the board.

## Power Supply

<table>
<tr><th>Supply</th><th>Source</th><th>Decoupling</th></tr>
<tr><td>AVDD (5V_AFE)</td><td>CN1</td><td>10µF + 100nF</td></tr>
<tr><td>DVDD (3V3_CLEAN)</td><td>CN2</td><td>10µF + 100nF</td></tr>
</table>

The analog and digital supplies enter on separate connectors and stay separate all the way to the chip, so digital switching noise does not couple into the analog supply.

## Clock

A **7.68MHz crystal** (X2) with **18pF** load capacitors drives the ADS1256 clock input. It sits right next to the chip with a surrounding guard ring and ground underneath to keep the oscillator stable and isolated from nearby signals.

## Digital Interface

The SPI lines map directly to the ESP32 VSPI pins. SCLK, DIN and DOUT each have a **100Ω series resistor** to damp ringing and reduce edge related noise.

<table>
<tr><th>ADS1256 Pin</th><th>ESP32 GPIO</th><th>Function</th></tr>
<tr><td>SCLK</td><td>IO18</td><td>SPI clock</td></tr>
<tr><td>DIN</td><td>IO23</td><td>SPI data in (MOSI)</td></tr>
<tr><td>DOUT</td><td>IO19</td><td>SPI data out (MISO)</td></tr>
<tr><td>CS</td><td>IO5</td><td>Chip select</td></tr>
<tr><td>DRDY</td><td>IO4</td><td>Data ready interrupt</td></tr>
<tr><td>RESET</td><td>IO16</td><td>Hardware reset</td></tr>
<tr><td>SYNC / PDWN</td><td>IO17</td><td>Synchronisation and power down</td></tr>
</table>

RESET and SYNC/PDWN are pulled up to 3.3V through **10kΩ** resistors, so the ADC stays active if the controller leaves them floating. A reset push button (SW1) with a 100nF debounce capacitor allows a manual reset.

The same interface is available on two connectors wired in parallel: a 9 pin JST ZR connector (U3) for a locking cable and a 1×9 2.54mm pin header (J1) for breadboards and jumper wires.

## PCB Design

**Stackup:** 4 layers

<table>
<tr><th>Layer</th><th>Function</th></tr>
<tr><td>L1 (Top)</td><td>ADS1256, input filters, clock, connectors and signal routing, with GND pour</td></tr>
<tr><td>L2</td><td>Continuous solid GND plane</td></tr>
<tr><td>L3</td><td>Power layer with separate pours for 5V_AFE, 3V3_CLEAN and the reference</td></tr>
<tr><td>L4 (Bottom)</td><td>Additional routing, with GND pour</td></tr>
</table>

**Layout highlights:**

1. Analog inputs enter on the left and digital SPI exits on the right, so the analog and digital signal paths never cross
2. Input filter components are grouped tightly next to the ADC input pins
3. Differential channel pairs are routed together as pairs
4. Dense ground stitching vias across the board and a via fence along the board edge
5. Clock crystal enclosed in a guard ring
6. Silkscreen on the back carries board identification and usage notes

## Key Components

<table>
<tr><th>Ref</th><th>Part</th><th>Function</th></tr>
<tr><td>U1</td><td>ADS1256IDBR</td><td>24 bit, 8 channel delta sigma ADC</td></tr>
<tr><td>X2</td><td>7.68MHz Crystal</td><td>ADC master clock</td></tr>
<tr><td>R3 to R10</td><td>301Ω</td><td>Input filter series resistors</td></tr>
<tr><td>C6, C7, C13, C15</td><td>100nF</td><td>Input filter capacitors</td></tr>
<tr><td>C9, C12, C14, C16</td><td>100pF</td><td>Input filter capacitors</td></tr>
<tr><td>R1, R2</td><td>49.9Ω</td><td>Reference filter resistors</td></tr>
<tr><td>C3</td><td>47µF</td><td>Reference bulk capacitor</td></tr>
<tr><td>R11 to R13</td><td>100Ω</td><td>SPI series termination</td></tr>
<tr><td>R14, R15</td><td>10kΩ</td><td>RESET and SYNC pull ups</td></tr>
<tr><td>SW1</td><td>Push button</td><td>Manual reset</td></tr>
<tr><td>CN1 to CN3</td><td>JST ZR 3 pin</td><td>5V_AFE, 3V3_CLEAN and reference inputs</td></tr>
<tr><td>CN4 to CN7</td><td>JST ZR 4 pin</td><td>Differential analog inputs</td></tr>
<tr><td>U3</td><td>JST ZR 9 pin</td><td>SPI and control connector</td></tr>
<tr><td>J1</td><td>1×9 2.54mm header</td><td>SPI and control header</td></tr>
</table>

## Current Status

<table>
<tr><th>Stage</th><th>Status</th></tr>
<tr><td>Schematic</td><td>Complete</td></tr>
<tr><td>PCB placement</td><td>Complete</td></tr>
<tr><td>Routing</td><td>Complete</td></tr>
<tr><td>Fabrication and assembly</td><td>Pending</td></tr>
<tr><td>Physical bring up and testing</td><td>Pending</td></tr>
</table>

## Next Steps

1. Fabricate and assemble the board
2. Power it from the Universal Precision Power Board and confirm AVDD, DVDD and the reference
3. Write ESP32 firmware to configure the ADS1256 and read data over SPI using DRDY
4. Measure noise with shorted inputs at different data rates and gains to find the real effective resolution
5. Complete the detailed project documentation

## Related Projects

1. Universal Precision Power Board (supplies 5V_AFE, 3V3_CLEAN and the 2.5V reference)
2. ESP32 Custom Development Board (controller)

## References

1. [ADS1256 Datasheet](https://www.ti.com/lit/ds/symlink/ads1256.pdf)
2. [ESP32 Technical Reference](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf)
3. JST ZR connector and other component datasheets sourced from LCSC through EasyEDA
