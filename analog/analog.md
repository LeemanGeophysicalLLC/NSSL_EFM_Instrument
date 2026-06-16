# Analog Charge Amplifier

The analog board performs coulomb counting on the signal from the field-mill electrode and
filters the result before passing it to the rotating electronics for digitization and logging.
The design closely follows the original instrument's analog chain. A small ATtiny
microcontroller generates a stable square-wave clock that sets the switched-capacitor filter's
cutoff frequency independent of temperature.

**Schematic revision:** 2.1 (2021-03-25)

---

## Functional Overview

| Stage | Component | Function |
|-------|-----------|----------|
| Charge amplifier | LF356 (U1) | Integrates charge from the electrode; 100 MΩ feedback resistor (R1) provides DC bleed |
| Protection | D1, D2 (1N4148) | Clamp the amplifier input against large transients |
| Anti-alias / noise filter | LTC1569-6 (U2) | 10th-order switched-capacitor elliptic lowpass; cutoff set by external clock |
| Filter clock source | ATtiny45 (U3) + 20 MHz crystal (Y1) | Generates a stable 50% duty-cycle square wave; temperature-independent |

---

## Filter Cutoff Frequency

The LTC1569-6 has a fixed clock-to-cutoff ratio of 100:1. The ATtiny generates the clock on
PB0 (Arduino pin 0) using `delayMicroseconds`:

```cpp
delayMicroseconds(333);  // HIGH half-period
delayMicroseconds(333);  // LOW half-period
```

| Parameter | Value |
|-----------|-------|
| Half-period | 333 µs |
| Clock period | 666 µs |
| Clock frequency | ~1502 Hz |
| Filter cutoff (f_CLK / 100) | **~15 Hz** |

To change the cutoff frequency, adjust the `delayMicroseconds` value. The relationship is:

```
f_cutoff (Hz) = 5000 / delay_us
```

| `delay_us` | Clock frequency | Cutoff frequency |
|------------|-----------------|-----------------|
| 250 | 2000 Hz | 20 Hz |
| 333 | 1502 Hz | 15 Hz (current) |
| 500 | 1000 Hz | 10 Hz |
| 1000 | 500 Hz | 5 Hz |

The 20 MHz crystal ensures the timing is stable across the operating temperature range of the
instrument — an RC-based oscillator would cause the cutoff frequency to drift with temperature.

---

## Firmware Details

The firmware (`Analog_Firmware.ino`) runs on the ATtiny45. It is minimal by design — setup
configures pin 0 as an output, and the loop toggles it continuously with no other logic.

**Target:** ATtiny45, any standard ATtiny Arduino core  
**Clock source:** External 20 MHz crystal (Y1) with 20 pF load capacitors (C13, C15)

**Fuse note:** The ATtiny must be configured to use the external crystal oscillator. The default
factory fuses use the internal RC oscillator; flashing the fuses to `CKSEL=1111` (external
crystal, full swing) is required for correct frequency output.

---

## Board-to-Board Connector (J1)

10-pin 2.54 mm pitch straight pin header. Pins 7–10 are not connected.

| Pin | Signal | Direction | Description |
|-----|--------|-----------|-------------|
| 1 | GND | — | Ground |
| 2 | +5V | In | Positive supply |
| 3 | −5V | In | Negative supply |
| 4 | In | In | Analog input from electrode / charge source |
| 5 | Out | Out | Filtered analog output to rotating electronics |
| 6 | GND | — | Ground |
| 7–10 | — | — | Not connected |

---

## Bill of Materials

| Reference | Value | Description |
|-----------|-------|-------------|
| U1 | LF356 | JFET-input op-amp, charge amplifier (SOIC-8) |
| U2 | LTC1569-6 | 10th-order switched-capacitor lowpass filter (SOIC-8) |
| U3 | ATtiny45-20SU | Filter clock generator (SOIC-8) |
| Y1 | 20 MHz | Crystal oscillator for ATtiny (HC-49/U vertical) |
| R1 | 100 MΩ | Charge amplifier feedback / DC bleed (1206) |
| R2 | 100 Ω | Series input resistor (0603) |
| R3 | 10 kΩ | Bias resistor (0603) |
| C7, C8 | 0.39 nF | Charge amplifier integration capacitors (0603) |
| C13, C15 | 20 pF | Crystal load capacitors (0603) |
| D1, D2 | 1N4148 | Input protection diodes (SOD-323F) |
| C1, C2, C5, C9, C11, C16 | 1 µF | Bulk decoupling (0603) |
| C3, C4, C6, C10, C12, C14 | 0.1 µF | High-frequency decoupling (0603) |
| J1 | 10-pin header | Board-to-board connector, 2.54 mm pitch |
| TP1–TP5 | Test point | SMD test points (not populated in production) |

---

## Firmware Checkout Procedure

### Clock Output Verification

| Step | Expected Outcome |
|------|-----------------|
| Program ATtiny fuses for external crystal oscillator | Fuse bits set correctly (CKSEL=1111) |
| Flash `Analog_Firmware.ino` to ATtiny45 | No errors reported |
| Probe PB0 (ATtiny pin 5) with an oscilloscope | Square wave present |
| Measure clock frequency | ~1502 Hz (666 µs period, 333 µs each phase) |
| Measure duty cycle | 50% ±2% |
| Confirm signal is stable over temperature | No significant frequency drift as board warms up |

### Filter Verification

| Step | Expected Outcome |
|------|-----------------|
| Apply ±5V and verify supply voltages at J1 pins 2 and 3 | +5V and −5V present |
| Probe LTC1569-6 clock input pin | Clock signal from ATtiny present |
| Inject a low-frequency sine wave (1–5 Hz) at J1 pin 4 | Signal passes through; visible at J1 pin 5 |
| Inject a signal above cutoff (~50 Hz) at J1 pin 4 | Signal significantly attenuated at J1 pin 5 |
| Measure −3 dB point of filter | Should be approximately 15 Hz |

### Charge Amplifier Verification

| Step | Expected Outcome |
|------|-----------------|
| Confirm 100 MΩ feedback resistor (R1) is populated | DC output settles to near 0V with no input |
| Apply a known charge pulse to J1 pin 4 | Output pulse visible and proportional |
| Confirm protection diodes D1, D2 do not conduct under normal signal levels | No clipping on small signals |
