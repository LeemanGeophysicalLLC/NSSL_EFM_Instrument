# Balloon Tracker

The tracker provides GPS position reports to instrument operators via the Iridium satellite
network. It transmits at a progressive rate — frequent updates early in flight when the balloon
is climbing and decisions matter most, tapering to hourly or bi-hourly updates during long cruise
phases to conserve battery. The unit also allocates a downlink receive buffer on each Iridium
session for future relay of cutdown commands via XBee.

## System Overview

| Component | Part | Interface |
|-----------|------|-----------|
| Microcontroller | ATmega328P (Arduino Pro Mini, 8 MHz) | — |
| GPS receiver | u-blox (NMEA, TinyGPS++) | SoftwareSerial, 9600 baud |
| Satellite modem | RockBLOCK / Iridium SBD | Hardware UART, 19200 baud |
| XBee radio | XBee (debug/cutdown relay) | SoftwareSerial, 9600 baud (debug build only) |
| Activity indicator | Single LED | Digital (pin 13) |

**Flashing:** `avrdude -p m328p -c avrispmkII -P usb -U flash:w:firmware.hex:i`

---

## Compile-Time Configuration

| Define | Default | Description |
|--------|---------|-------------|
| `ENABLE_IRIDIUM` | defined | Enable actual Iridium SBD transmissions. Comment out to test without incurring airtime costs. |
| `ENABLE_DEBUG` | not defined | Route debug messages and Iridium payload previews to the XBee serial port. |
| `DIAGNOSTICS` | `false` | Enable IridiumSBD library internal diagnostic callbacks. |

**For flight:** `ENABLE_IRIDIUM` must be defined. `ENABLE_DEBUG` should be left undefined.

---

## Unit ID

The unit ID is hardcoded at the top of `main.cpp`:

```cpp
uint8_t unit_id = 16;
```

This value is included in every transmitted message. It must be set to the correct instrument
number before flashing a new unit.

---

## Transmission Cadence

The tracker uses a five-tier rate ladder based on logical elapsed time since boot. Logical time
tracks actual elapsed time including deep-sleep periods, so the schedule is based on mission
elapsed time rather than wall clock.

| Mission Phase | Logical Time Range | TX Interval |
|---------------|--------------------|-------------|
| Launch / early ascent | 0 – 10 min | 1 min |
| Mid ascent | 10 – 60 min | 5 min |
| Late ascent / float entry | 60 – 120 min | 15 min |
| Cruise | 120 – 300 min (~5 h) | 60 min |
| Extended cruise | > 300 min | 120 min |

The modem is put to sleep between transmissions when the interval is 15 minutes or longer,
and woken 15 seconds before the next scheduled TX window to allow it to settle.

---

## Power Management

Between transmissions the MCU enters a deep-sleep (AVR POWER_DOWN) using the Adafruit SleepyDog
library. Sleep cycles are ~3 seconds long. During each cycle the firmware:

1. Parses ~200 ms of GPS data.
2. Sleeps the MCU for ~3 s (`Watchdog.sleep(3000)`).
3. Adds the actual sleep duration to a `slept_accum_ms` accumulator used for logical time.

In the 15-second pre-TX window the MCU stays fully awake and continuously parses GPS to ensure
the fix is current before transmitting.

---

## Transmit Message Format

Each Iridium SBD message is a plain ASCII string, max 65 bytes:

```
<millis>,<unit_id>,<lat>,<lon>,<alt>\r
```

| Field | Format | Description |
|-------|--------|-------------|
| `millis` | `%lu` | `millis()` value at TX time (ms since boot, not logical time) |
| `unit_id` | `%d` | Instrument unit ID (hardcoded in firmware) |
| `lat` | `%.4f` | Latitude in decimal degrees (4 decimal places) |
| `lon` | `%.4f` | Longitude in decimal degrees (4 decimal places) |
| `alt` | `%.1f` | Altitude in meters (1 decimal place) |

Example: `12345678,16,35.1234,-97.5678,1234.5\r`

If the GPS has no valid fix at TX time, TinyGPS++ returns its default invalid values (0.0 for
lat/lon, 0.0 for altitude).

---

## Firmware Details

### GPS Configuration

At startup, the GPS is configured for high-altitude operation via a raw UBX command (NAV5, dynamic
model 6 — Airborne <1g). The firmware retries until an ACK is received or a 10-second timeout
expires. Note: ACK detection uses SoftwareSerial and may be unreliable; the command is assumed
to have taken effect if no timeout occurs.

GPS baud rate is 9600. The PPS pin (pin 6) is configured as an input but is not used for
interrupt-driven timing in this firmware.

### Iridium Modem Control

| Action | Mechanism |
|--------|-----------|
| Power on | `PIN_RB_SLEEP` set to high-impedance input (modem self-powers) |
| Power off / sleep | `PIN_RB_SLEEP` driven LOW (active-low sleep) |
| Network status | `PIN_RB_NETWORK` read as input (not currently acted on) |
| TX | `modem.sendSBDText()` — implicitly checks the downlink mailbox per SBD protocol |

If `modem.begin()` fails at startup, the LED flashes rapidly (10 Hz, 50 cycles) and the MCU
performs a software reset.

### LED Behavior

| Pattern | Meaning |
|---------|---------|
| 1 Hz blink (1 s on, 1 s off) | GPS warm-up during 60 s startup wait |
| 10 Hz rapid flash (50 cycles) | Iridium modem init failure — unit will reset |
| 20 ms pulse every 5 s | Normal operation heartbeat |
| 60 ms pulse after TX | Transmission confirmed |

---

## Pin Assignments

| Pin | Name | Direction | Description |
|-----|------|-----------|-------------|
| 0 | `PIN_MODEM_SERIAL_RX` | RX | Hardware UART RX from Iridium modem |
| 1 | `PIN_MODEM_SERIAL_TX` | TX | Hardware UART TX to Iridium modem |
| 2 | `PIN_GPS_SERIAL_TX` | TX | SoftwareSerial TX to GPS |
| 3 | `PIN_GPS_SERIAL_RX` | RX | SoftwareSerial RX from GPS |
| 4 | `PIN_XBEE_SERIAL_RX` | RX | SoftwareSerial RX from XBee (debug build only) |
| 5 | `PIN_XBEE_SERIAL_TX` | TX | SoftwareSerial TX to XBee (debug build only) |
| 6 | `PIN_GPS_PPS` | Input | GPS PPS signal (wired, not used in firmware) |
| 7 | `PIN_RB_NETWORK` | Input | Iridium network available indicator |
| 8 | `PIN_RB_SLEEP` | Input/Output | Iridium modem sleep control (high-Z = on, LOW = sleep) |
| 13 | `PIN_LED_ACTIVITY` | Output | Activity LED |

---

## Firmware Checkout Procedure

### Pre-Flight Setup

| Step | Expected Outcome |
|------|-----------------|
| Verify `unit_id` in `main.cpp` matches the instrument number | Correct ID present in source |
| Confirm `ENABLE_IRIDIUM` is defined | Iridium transmissions will be active in flight |
| Confirm `ENABLE_DEBUG` is not defined | No XBee debug traffic during flight |
| Flash firmware via avrdude | No errors reported |

### Power-On and GPS Lock

| Step | Expected Outcome |
|------|-----------------|
| Apply power | LED begins 1 Hz blink |
| Wait up to 60 s | LED continues blinking during GPS warm-up |
| After 60 s, if Iridium init succeeds | LED goes dark (normal operation begins) |
| If LED flashes rapidly (~10 Hz) then stops | Iridium modem init failed — check modem connections and retry |

### Transmission Verification

| Step | Expected Outcome |
|------|-----------------|
| After first 1-minute interval, observe LED | 60 ms confirmation pulse after TX |
| Check Iridium inbox (Rock7 portal or equivalent) | Message received in format `millis,unit_id,lat,lon,alt` |
| Verify `unit_id` in received message | Matches the value flashed to the unit |
| Verify lat/lon/alt are plausible | GPS had a valid fix at TX time |
| Allow unit to run for 15 minutes and verify rate change | TX interval increases to 5 min after 10 min elapsed |

### Cleanup

After ground testing, confirm the unit is transmitting at the correct rate before flight. If
testing with `ENABLE_IRIDIUM` commented out, reflash with it defined before deployment.
