# Orientation Paddle

The orientation paddle contains an IMU, magnetic rotary encoder, and GPS receiver that together
log the instrument's attitude and rotation at 10 Hz. GPS-synchronized timestamps allow direct
comparison with data recorded in the field-mill spheres and the motor controller. All data are
written to a microSD card.

## System Overview

| Component | Part | Interface |
|-----------|------|-----------|
| Microcontroller | STM32F103C8 (Blue Pill, 128 KB flash) | — |
| IMU | ICM-20948 (accel, gyro, magnetometer) | I2C |
| Rotary encoder | AS5600 magnetic angle sensor | I2C |
| GPS receiver | u-blox GNSS module | I2C + PPS interrupt |
| Storage | microSD card | SPI |
| Status indicator | RGB LED | PWM (PB8, PB9, PA8) |

---

## LED Status Indicators

| Color | Meaning |
|-------|---------|
| Blue (solid) | Waiting for GPS lock |
| Green (solid) | Logging — GPS locked and data flowing |
| Red (solid) | Error — sensor, GPS, or SD card failure |
| RGB cycle on power-up | Normal boot sequence (red → green → blue → off) |

---

## Operating States

The firmware runs a simple three-state machine:

| State | Description |
|-------|-------------|
| `STATE_ACQUIRING_GPS` | Startup state; waiting for a valid GPS fix and PPS sync before logging begins |
| `STATE_LOGGING` | Normal operation; sensors sampled every ~66 ms, data buffered and flushed to SD |
| `STATE_ERROR` | Entered on any sensor init failure, GPS timeout, or SD write error; LED turns red and system halts on startup errors |

---

## Firmware Details

### Timing and Sampling

| Parameter | Value |
|-----------|-------|
| Sample interval | 66 ms (~15 Hz) |
| Write threshold | Every 20 buffered lines (flush to SD) |
| Maximum buffer depth | 30 lines |
| GPS navigation rate | 1 Hz |
| GPS PPS pulse width | 100 ms |
| Watchdog timeout | 26 s |
| Debug UART baud | 115200 |
| GPS UART baud | 115200 |
| SPI clock (SD) | 18 MHz |
| I2C clock | 400 kHz |

### IMU Configuration

The ICM-20948 is initialized with the following full-scale ranges:

| Sensor | Full-Scale Range |
|--------|-----------------|
| Accelerometer | ±4 g |
| Gyroscope | ±1000 °/s |
| Magnetometer | Factory default (AK09916 internal) |

### GPS Configuration

The u-blox receiver is configured at startup and settings are saved to battery-backed memory:

| Setting | Value |
|---------|-------|
| Dynamic model | AIRBORNE <4g> |
| Navigation frequency | 1 Hz |
| UART1 baud rate | 115200 |
| Enabled NMEA sentences | RMC, ZDA, GGA |
| Disabled NMEA sentences | GLL, GSA, GSV, VTG, GNS |
| Time pulse (PPS) | 1 Hz, 100 ms active-high pulse |

If the GPS module is not detected on the I2C bus at startup, the firmware enters `STATE_ERROR` and halts.

---

## Log File Format

Log files are named `OPLOGnnn.TXT` (e.g., `OPLOG000.TXT`) and are created sequentially; a new
file is opened on each power cycle. Files are plain-text CSV.

### CSV Header

```
Timestamp_UTC,time_quality,Millis_ms,Lat_degE7,Lon_degE7,Elev_mm,Angle_mdeg,AccX_ug,AccY_ug,AccZ_ug,GyroX_mdps,GyroY_mdps,GyroZ_mdps,MagX_nT,MagY_nT,MagZ_nT
```

### Column Definitions

| Column | Units | Type | Description |
|--------|-------|------|-------------|
| `Timestamp_UTC` | — | string | `YYYYMMDDTHHmmSS.mmm` — GPS-synchronized UTC timestamp derived from PPS anchor + millis offset |
| `time_quality` | — | int | `1` = GPS locked at time of sample; `0` = no lock or pre-lock |
| `Millis_ms` | ms | uint32 | `millis()` value at time of sample (relative to boot) |
| `Lat_degE7` | deg × 10⁷ | int32 | Latitude from GPS (−999×10⁷ if not valid) |
| `Lon_degE7` | deg × 10⁷ | int32 | Longitude from GPS (−999×10⁷ if not valid) |
| `Elev_mm` | mm | int32 | Altitude from GPS in mm (−999000 if not valid) |
| `Angle_mdeg` | m° | int32 | AS5600 rotary encoder angle × 1000; 0–360000 |
| `AccX_ug` | μg | int32 | IMU accelerometer X × 1000 (milli-g) |
| `AccY_ug` | μg | int32 | IMU accelerometer Y × 1000 (milli-g) |
| `AccZ_ug` | μg | int32 | IMU accelerometer Z × 1000 (milli-g) |
| `GyroX_mdps` | m°/s | int32 | IMU gyroscope X × 1000 |
| `GyroY_mdps` | m°/s | int32 | IMU gyroscope Y × 1000 |
| `GyroZ_mdps` | m°/s | int32 | IMU gyroscope Z × 1000 |
| `MagX_nT` | nT | int32 | IMU magnetometer X × 1000 |
| `MagY_nT` | nT | int32 | IMU magnetometer Y × 1000 |
| `MagZ_nT` | nT | int32 | IMU magnetometer Z × 1000 |

**Note:** All float sensor values are converted to integer fixed-point (×1000) before logging to
avoid floating-point formatting overhead on the STM32. A reading of `−9999000` in any sensor
column indicates a failed read for that sample.

### Timestamp Format

When GPS is locked the timestamp follows `YYYYMMDDTHHmmSS.mmm`. Before the first GPS fix is
acquired, the timestamp field is written as `00000000T000000.000` and `time_quality` is `0`.

---

## Pin Assignments

| Pin | Function |
|-----|----------|
| PB8 | LED Red (PWM) |
| PB9 | LED Green (PWM) |
| PA8 | LED Blue (PWM) |
| PA4 | SD card CS (SPI) |
| PB15 | SD card card-detect |
| PA1 | GPS PPS interrupt |
| PA2 | GPS UART TX (SerGPS) |
| PA3 | GPS UART RX (SerGPS) |
| PB10 | Debug UART TX (SerDebug) |
| PB11 | Debug UART RX (SerDebug) |

---

## Firmware Checkout Procedure

This checklist verifies proper operation after any firmware modification.

### Power-On Self-Test

| Step | Expected Outcome |
|------|-----------------|
| Apply power | LED cycles red → green → blue → off (approximately 1 s each) |
| Wait for GPS lock (up to ~60 s) | LED turns green once lock is acquired |
| If LED stays blue past 60 s | No GPS signal — check antenna connection |
| If LED turns red immediately | Hardware init failed — check SD card and sensor connections |

### GPS Lock and Timestamp Verification

| Step | Expected Outcome |
|------|-----------------|
| Allow unit to acquire GPS lock | LED turns green |
| Remove SD card and open most recent `OPLOGnnn.TXT` | File exists with CSV header |
| Inspect first rows logged before lock | `time_quality` = 0, timestamp = `00000000T000000.000` |
| Inspect rows logged after lock | `time_quality` = 1, timestamp is a valid UTC value |
| Verify timestamp advances at correct rate | Successive rows differ by ~66 ms in `Millis_ms` |

### Sensor Data Verification

| Step | Expected Outcome |
|------|-----------------|
| With unit at rest, inspect `AccZ_ug` | Should read approximately 1000000 (1 g) |
| With unit at rest, inspect gyro columns | Should read near 0 |
| Rotate the paddle slowly by hand | `Angle_mdeg` column should change monotonically 0–360000 |
| Inspect magnetometer columns | Non-zero values; vary with orientation change |
| No column should read `−9999000` during normal operation | If seen, a sensor read failed |

### SD Card Write Verification

| Step | Expected Outcome |
|------|-----------------|
| Let unit log for at least 2 minutes | `OPLOGnnn.TXT` should grow in size |
| Power off normally, then remove SD card | File is not corrupted or truncated |
| Verify line count | Expect approximately 900 rows per minute at ~15 Hz (66 ms interval) |

### Cleanup

After firmware testing, verify GPS re-locks on the next power cycle before returning the unit to service.

---

**Note:** This documentation matches firmware version 1.0. Enable `DEBUG_PRINT_ENABLE` in `main.cpp` and connect to SerDebug (PB10/PB11) at 115200 baud for verbose startup and runtime diagnostics.
