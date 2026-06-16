# Rotating Electronics

The rotating board is the primary data acquisition system for the instrument. It digitizes the
analog output from the charge amplifier at high rate, reads a 9-DOF IMU and environmental sensor,
tracks GPS position, and logs everything to a microSD card. Two record types are interleaved in
the log: high-frequency ADC + IMU samples at 50 Hz and low-frequency GPS + environmental
samples at 1 Hz.

---

## Firmware Variants

The repository contains several firmware builds. Only `original operational` is the current
flight firmware.

| Directory | Purpose |
|-----------|---------|
| `original operational` | **Flight firmware** — full sensor suite, SD card logging, GPS-synchronized timing |
| `nosd` | Development variant — SD removed; data output over serial to an external OpenLog |
| `nssl_custom_openlog` | Paired firmware for an OpenLog board used with the `nosd` variant |
| `simple bringup` | ADC-only bench test; used to verify ADS131M04 communication during hardware bring-up |
| `OpenLog_Custom` | Stripped SparkFun OpenLog firmware (no CLI, increased RX buffer) for use as a serial logger |

**For flight:** use `original operational`.

---

## System Overview

| Component | Part | Interface |
|-----------|------|-----------|
| Microcontroller | STM32F103C8 (Blue Pill, 128 KB flash) | — |
| ADC | ADS131M04 (4-channel, 24-bit) | SPI |
| IMU | ICM-20948 (accel, gyro, magnetometer) | I2C |
| Environmental sensor | BME680 (temp, pressure, humidity) | I2C (addr 0x76) |
| GPS receiver | u-blox GNSS module | I2C + UART + PPS interrupt |
| Storage | microSD card | SPI |
| Status indicator | RGB LED | PWM (PB8, PB9, PA8) |
| Debug / radio port | UART | PB10/PB11 |
| XBee port | UART | PA9/PA10 |

---

## Compile-Time Flags

| Define | Default | Description |
|--------|---------|-------------|
| `WAIT_ON_GPS_FIX` | defined | Block in `setup()` until GPS lock is acquired before beginning logging. Comment out for bench testing without GPS. |
| `WATCHDOG_ENABLE` | not defined | Enable the STM32 independent watchdog (26 s timeout). Should be defined for flight. |
| `DEBUG_PRINT_ENABLE` | not defined | Route verbose diagnostic messages to the debug UART (PB10/PB11) at 115200 baud. |

**For flight:** define `WATCHDOG_ENABLE`. Leave `WAIT_ON_GPS_FIX` defined. Leave `DEBUG_PRINT_ENABLE` undefined.

---

## Data Acquisition

### ADC (ADS131M04)

The ADS131M04 is a 4-channel simultaneous-sampling 24-bit ADC. It is driven by interrupt on
the DRDY line, which fires at 250 Hz. The ISR decimates by 5 to produce 50 Hz samples passed
to the main loop.

| Parameter | Value |
|-----------|-------|
| Raw DRDY rate | 250 Hz |
| Decimation factor | 5 |
| Effective sample rate | **50 Hz** |
| Oversampling ratio | OSR_16384 |
| PGA gain (all channels) | 1× |
| Input mux (all channels) | AIN0P / AIN0N (differential) |

All four ADC channels are enabled and logged in every HF record, though in current hardware only
some channels carry signals from the analog paddle.

### IMU (ICM-20948)

The IMU is read once per 50 Hz ADC sample in the main loop. Gyroscope data is captured in the
struct but currently commented out of the log output; only accelerometer and magnetometer are
written to the log.

| Sensor | Logged | Units in log |
|--------|--------|--------------|
| Accelerometer (X, Y, Z) | Yes | mg (×1000 of g) |
| Gyroscope (X, Y, Z) | No (captured, not written) | — |
| Magnetometer (X, Y, Z) | Yes | µT (1× scale, integer) |

### Environmental Sensor (BME680)

Sampled once per second alongside GPS. Gas heater is disabled; only temperature, pressure, and
humidity are used.

| Channel | Oversampling | Units in log |
|---------|-------------|--------------|
| Temperature | 8× | °C × 10 (int16) |
| Pressure | 4× | hPa × 10 (int16) |
| Humidity | 2× | %RH × 10 (int16) |

A failed BME read writes `INT16_MIN` (−32768) to all three fields.

---

## GPS and Timing

The u-blox receiver is configured at startup (settings saved to battery-backed memory):

| Setting | Value |
|---------|-------|
| Dynamic model | AIRBORNE <2g> |
| Navigation frequency | 1 Hz |
| UART1 baud rate | 115200 |
| Enabled NMEA sentences | RMC, ZDA, GGA |
| Time pulse (PPS) | 1 Hz, 100 ms active-high pulse |

GPS initialization begins with a factory reset (`factoryReset()` + `factoryDefault()`), so
settings are re-applied on every boot.

### PPS Synchronization

The PPS rising edge fires an ISR (`ppsISR`) which captures `millis()` and resets the 50 Hz
sample index counter (`hfIdx = 0`). Each HF record stores:

- `pps_utc` — the raw GPS time value from TinyGPS++ at the last valid sentence
- `ms_offset` — which 20 ms grid slot within the current second the sample belongs to (`idx × 20`)
- `processor_ms` — the raw `millis()` value when the record was built

---

## Log File Format

### File Naming

Log files are named `MMDDHHMM.TXT` based on GPS time, where the minute is rounded down to the
nearest 10-minute boundary. A new file is opened every 10 minutes. Each file is pre-allocated
to 4 MB on creation to reduce SD fragmentation. If GPS lock has not yet been acquired the file
is named `00000000.TXT`.

### Record Types

Two record types are interleaved in each file, identified by the first field.

#### HF Record (type `1`) — 50 Hz

```
1,<pps_utc>,<ms_offset>,<processor_ms>,<ch0>,<ch1>,<ch2>,<ch3>,<ax>,<ay>,<az>,<mx>,<my>,<mz>
```

| Field | Type | Description |
|-------|------|-------------|
| `1` | literal | Record type identifier |
| `pps_utc` | uint32 | GPS UTC time value from TinyGPS++ (format: HHMMSSCC) |
| `ms_offset` | uint32 | Milliseconds since last PPS (0, 20, 40, … 980) |
| `processor_ms` | uint32 | `millis()` at time of record |
| `ch0`–`ch3` | uint32 | Raw ADS131M04 24-bit ADC counts (4 channels) |
| `ax`, `ay`, `az` | int16 | Accelerometer in mg (g × 1000) |
| `mx`, `my`, `mz` | int16 | Magnetometer in µT (integer) |

#### LF Record (type `2`) — 1 Hz

```
2,<pps_utc>,<ms_offset>,<processor_ms>,<lat>,<lon>,<alt>,<temp>,<pressure>,<humidity>
```

| Field | Type | Description |
|-------|------|-------------|
| `2` | literal | Record type identifier |
| `pps_utc` | uint32 | GPS UTC time value from TinyGPS++ |
| `ms_offset` | uint32 | Milliseconds since last PPS |
| `processor_ms` | uint32 | `millis()` at time of record |
| `lat` | int32 | Latitude × 10⁵ (decimal degrees) |
| `lon` | int32 | Longitude × 10⁵ (decimal degrees) |
| `alt` | uint16 | Altitude in whole metres (rounded) |
| `temp` | int16 | Temperature in °C × 10 (`INT16_MIN` on error) |
| `pressure` | int16 | Pressure in hPa × 10 (`INT16_MIN` on error) |
| `humidity` | int16 | Humidity in %RH × 10 (`INT16_MIN` on error) |

### Write Behavior

The circular log buffer holds up to 128 `LogEntry` records. `flushLog()` drains up to 32
entries per call and flushes to the SD card at ~10 Hz. GPS is polled every 8 records during
drain to avoid starving the NMEA parser.

---

## LED Status Indicators

| Color | Meaning |
|-------|---------|
| RGB cycle on power-up | Normal boot sequence (red → green → blue → off, 1 s each) |
| Blue (solid) | Waiting for GPS lock |
| Green (solid) | Logging normally |
| Red (blinking) | Setup failed — sensor init or SD card error |
| Red (solid) | Runtime error — SD write failure |

---

## Pin Assignments

| Pin | Name | Direction | Description |
|-----|------|-----------|-------------|
| PA0 | `PIN_ADC_CS` | Output | ADS131M04 SPI chip select |
| PA1 | `PIN_GPS_PPS` | Input | GPS 1PPS interrupt |
| PA2 | `PIN_GPS_RX` | RX | GPS UART RX (SerGPS) |
| PA3 | `PIN_GPS_TX` | TX | GPS UART TX (SerGPS) |
| PA4 | `PIN_ADC_DRDY` | Input | ADS131M04 data-ready interrupt |
| PA5 | `PIN_SPI1_SCLK` | Output | SPI clock (ADC + SD) |
| PA6 | `PIN_SPI1_MISO` | Input | SPI MISO |
| PA7 | `PIN_SPI1_MOSI` | Output | SPI MOSI |
| PA8 | `PIN_LED_BLUE` | Output | Blue LED (PWM) |
| PA9 | `PIN_XBEE_TX` | TX | XBee UART TX |
| PA10 | `PIN_XBEE_RX` | RX | XBee UART RX |
| PA15 | `PIN_ADC_RESET` | Output | ADS131M04 hardware reset |
| PB4 | `PIN_SD_CD` | Input | SD card-detect |
| PB5 | `PIN_SD_CS` | Output | SD card SPI chip select |
| PB6 | `PIN_I2C_SCL` | Output | I2C clock (IMU, BME680, GPS) |
| PB7 | `PIN_I2C_SDA` | I/O | I2C data |
| PB8 | `PIN_LED_RED` | Output | Red LED (PWM) |
| PB9 | `PIN_LED_GREEN` | Output | Green LED (PWM) |
| PB10 | `PIN_RADIO_RX` | RX | Debug / radio UART RX |
| PB11 | `PIN_RADIO_TX` | TX | Debug / radio UART TX |

---

## Firmware Checkout Procedure

### Power-On Self-Test

| Step | Expected Outcome |
|------|-----------------|
| Apply power | LED cycles red → green → blue → off (1 s each) |
| LED turns blue after boot sequence | Waiting for GPS lock — all sensors initialized correctly |
| LED turns red and blinks rapidly | Setup failed — check sensor connections and SD card |

### Sensor Initialization Verification

Enable `DEBUG_PRINT_ENABLE` and connect to the debug UART (PB10/PB11) at 115200 baud to see
per-sensor init results during boot.

| Check | Expected Debug Output |
|-------|-----------------------|
| ADC | `Initializing ADC... ok` |
| IMU | `Initializing IMU... ok` |
| BME680 | `Initializing BME688... ok` |
| GPS | `Initializing GPS... ok` |
| SD init | `Initializing SD card... ok` |
| SD write test | `Testing SD write... ok` |

### GPS Lock and Logging

| Step | Expected Outcome |
|------|-----------------|
| Wait for GPS fix (outdoors or with antenna) | LED turns green |
| Remove SD card and inspect file listing | File named `MMDDHHMM.TXT` present (or `00000000.TXT` if no lock) |
| Open log file in text editor | Mix of lines starting with `1,` and `2,` |
| Count `1,` lines per second | Should be approximately 50 |
| Count `2,` lines per second | Should be approximately 1 |

### Data Quality Checks

| Check | Expected Outcome |
|-------|-----------------|
| `alt` field in LF records | Plausible altitude in metres for the location |
| `temp` / `pressure` / `humidity` | Not `−32768` (which indicates a BME read failure) |
| `ax`, `ay`, `az` in HF records with unit at rest | One axis near ±1000 (1 g), other two near 0 |
| `ms_offset` sequence in HF records | Steps of 20 ms (0, 20, 40, … 980), then resets |
| `pps_utc` advances by 1 second per group of 50 HF records | Confirms PPS synchronization |

### File Rollover Verification

| Step | Expected Outcome |
|------|-----------------|
| Allow unit to log across a 10-minute boundary | New `MMDDHHMM.TXT` file created |
| Inspect previous file | Closed cleanly; no truncated records |
| Each file pre-allocated to 4 MB | Visible as file size on SD; unfilled space is zeroed |

### Cleanup

After testing with `DEBUG_PRINT_ENABLE` or `WATCHDOG_ENABLE` disabled, reflash the flight
build with both flags in their correct flight state before deployment.
