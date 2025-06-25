# Motor Control

## Serial Command Reference

Commands are issued via serial terminal. All inputs are plain ASCII and case-insensitive. 
Parameters are space-separated. All commands must be followed by `\r\n`. Valid commands are 
acknowledged with `OK`. Invalid commands are acknowledged with `!` and may include an error 
message. All connections are to be made at 9600 baud through the on-board mini-USB port.

### Settings Control

| Command                      | Description                                         | Valid Range           | Example             |
|------------------------------|-----------------------------------------------------|------------------------|---------------------|
| `SETRPM <rpm - integer>`     | Set the PID target RPM                             | 180-240                 | `SETRPM 120`        |
| `SETKP <value - float>`      | Set the PID Kp value                               | 0.0–100.0              | `SETKP 0.5`         |
| `SETKI <value - float>`      | Set the PID Ki value                               | 0.0–100.0              | `SETKI 0.07`        |
| `SETKD <value - float>`      | Set the PID Kd value                               | 0.0–100.0              | `SETKD 0.0`         |
| `SETCURRENTLIM <mA - int>`   | Set motor current limit in mA                      | 10–3000               | `SETCURRENTLIM 300` |
| `SETRESTART <0|1>`           | Enable/disable auto-restart after stall detection  | 0 = off, 1 = on        | `SETRESTART 1`      |

### Diagnostics and Output

| Command   | Description                            |
|-----------|----------------------------------------|
| `SHOW`    | Print all current configuration values |
| `DUMP`    | Dump logged data in reverse order      |

### Maintenance

| Command       | Description                                 |
|---------------|---------------------------------------------|
| `RESETCONFIG` | Reset FRAM settings to factory defaults     |

---

## Firmware Details

### FRAM Memory Map (0x0000 – 0x03E7 Reserved)

The first 1000 bytes of FRAM are reserved for settings and persistent metadata.

| Offset | Size | Name                | Type       | Description                                      |
|--------|------|---------------------|------------|--------------------------------------------------|
| 0x0000 | 4    | Magic Number        | `uint32_t` | Constant value (e.g., `0xEFABEFAB`) used to      |
|        |      |                     |            | detect uninitialized memory                      |
| 0x0004 | 2    | Format Version      | `uint16_t` | Structure version, for future compatibility      |
| 0x0006 | 2    | Power Cycle Count   | `uint16_t` | Incremented on each boot                         |
| 0x0008 | 2    | Log Head Index      | `uint16_t` | Index of the next available log entry            |
| 0x000A | 2    | Current Limit (mA)  | `uint16_t` | Maximum allowed motor current before shutdown    |
| 0x000C | 4    | PID Kp              | `float`    | Proportional gain                                |
| 0x0010 | 4    | PID Ki              | `float`    | Integral gain                                    |
| 0x0014 | 4    | PID Kd              | `float`    | Derivative gain                                  |
| 0x0018 | 2    | Setpoint RPM        | `uint16_t` | Target RPM for the PID controller                |
| 0x001A | 1    | Restart Enabled     | `uint8_t`  | 0 = disable automatic restart, 1 = enable        |
| 0x001B | —    | Reserved            | —          | Remaining bytes (0x001B–0x03E7) reserved for     |
|        |      |                     |            | future use                                       |

---

### Log Entry Format (starting at 0x03E8)

Each log entry is 16 bytes and recorded approximately every 10 seconds.

| Field                 | Size | Type        | Description                                      |
|-----------------------|------|-------------|--------------------------------------------------|
| Timestamp (s)         | 4    | `uint32_t`  | Seconds since boot                               |
| RPM                   | 2    | `int16_t`   | Current instrument RPM                           |
| Current (mA)          | 2    | `uint16_t`  | Motor current draw                               |
| Temperature (degC×10) | 2    | `int16_t`   | Controller temperature in degrees Celsius × 10   |
| Battery (mV)          | 2    | `uint16_t`  | Battery voltage                                   |
| Flags                 | 1    | `uint8_t`   | Fault or status flags (reserved for future use)  |
| Reserved              | 3    | —           | Reserved/padding                                 |

## Firmware Checkout Procedure
This checklist can be used to verify proper operation and no regressions after the device firmware has been modified.


Tests are designed to verify that each serial command behaves correctly, including:
- Argument count validation
- Range enforcement
- Settings persistence
- Parser robustness to malformed input

### Test Outcome Legend

- (success expected) → Command should return `OK`
- (failure expected) → Command should return `!`
- (observe) → No crash or instability; parser must remain functional

### SETRPM <value> (Valid range: 60–300)

| Test Case | Expected Outcome |
|-----------|------------------|
| `SETRPM` | (failure expected) – too few arguments |
| `SETRPM 100 200` | (failure expected) – too many arguments |
| `SETRPM 20` | (failure expected) – below valid range |
| `SETRPM 400` | (failure expected) – above valid range |
| `SETRPM 150` | (success expected) – valid RPM set |
| Power cycle, then `SHOW` | (success expected) – verify `setpoint_rpm = 150` |
| `SETRPM qwertyuiopasdfghjklzxcvbnm1234567890` | (failure expected) – invalid input format |

### SETKP, SETKI, SETKD <value> (Valid range: 0–100)

Each command should be tested independently. Example below is for `SETKP`.

| Test Case | Expected Outcome |
|-----------|------------------|
| `SETKP` | (failure expected) – too few arguments |
| `SETKP 1 2` | (failure expected) – too many arguments |
| `SETKP -1` | (failure expected) – below valid range |
| `SETKP 101` | (failure expected) – above valid range |
| `SETKP 0.05` | (success expected) – gain accepted |
| Power cycle, then `SHOW` | (success expected) – verify value persisted |
| `SETKP ???!!!@@@###` | (failure expected) – invalid input format |

Repeat for:
- `SETKI`
- `SETKD`

### SETCURRENTLIM <milliamps> (Valid range: 100–1000)

| Test Case | Expected Outcome |
|-----------|------------------|
| `SETCURRENTLIM` | (failure expected) – too few arguments |
| `SETCURRENTLIM 1 2 3` | (failure expected) – too many arguments |
| `SETCURRENTLIM 50` | (failure expected) – below range |
| `SETCURRENTLIM 1500` | (failure expected) – above range |
| `SETCURRENTLIM 500` | (success expected) – valid current limit |
| Power cycle, then `SHOW` | (success expected) – verify value persisted |
| `SETCURRENTLIM spamspamspamspam` | (failure expected) – non-numeric input |

### SETCUTOFF <0|1>

| Test Case | Expected Outcome |
|-----------|------------------|
| `SETCUTOFF` | (failure expected) – too few arguments |
| `SETCUTOFF 1 2` | (failure expected) – too many arguments |
| `SETCUTOFF 2` | (failure expected) – invalid value |
| `SETCUTOFF -1` | (failure expected) – invalid value |
| `SETCUTOFF 0` | (success expected) – disable cutoff |
| Power cycle, then `SHOW` | (success expected) – verify persisted |
| `SETCUTOFF 1` | (success expected) – enable cutoff |
| Power cycle, then `SHOW` | (success expected) – verify persisted |
| `SETCUTOFF lolnope` | (failure expected) – non-numeric input |

### SETRESTART <0|1>

Same validation process as `SETCUTOFF`.

### SHOW

| Test Case | Expected Outcome |
|-----------|------------------|
| `SHOW` | (success expected) – prints current settings in readable format |

### DUMPLOG

| Test Case | Expected Outcome |
|-----------|------------------|
| `DUMPLOG` | (success expected) – prints data log (empty or not) |

### RESETCONFIG

| Test Case | Expected Outcome |
|-----------|------------------|
| `RESETCONFIG` | (success expected) – settings reset to defaults, log cleared |
| Power cycle, then `SHOW` | (success expected) – default settings confirmed |
| Verify `power_cycle_count == 1` | (success expected) – after reset |

### HELP

| Test Case | Expected Outcome |
|-----------|------------------|
| `HELP` | (success expected) – lists available commands and usage |


### Junk Input Stress Test

| Test Case | Expected Outcome |
|-----------|------------------|
| `AJSDHASKJDH1238127312##@@!!@!!()()()()_+_)(*&^%$#@!~` | (observe) – no crash; next command works |
| `LONGSTRINGOFNONCOMMANDSANDCHARSANDNUMBERS1234567890...` | (observe) – no buffer overflow |
| Send partially typed command | (observe) – system should not crash or become unresponsive |

### Power Cycle Verification

After each setting command that returns `OK`:

| Step | Expected Outcome |
|------|------------------|
| Power off and wait 10 seconds | - |
| Power on and run `SHOW` | (success expected) – changed setting persisted |
| Confirm `power_cycle_count` incremented | (success expected) |

### Over Current Protection and Restart Functionality
| Step | Expected Outcome |
|------|------------------|
| With the current limit set to 100 mA stall the motor and verify shutdown executes | (success expected) |
| With the restart function enabled the motor should restart in approximately 1 minute | (success expected) |
| With the restart function disabled the system should stay shutdown - wait for 5 minutes to verify | (success expected) |

## RPM Verification
| Step | Expected Outcome |
|------|------------------|
| Set the RPM to 210 and verify speed is reached within 60 seconds and maintained | (success expected) |
| Decrease the voltage driving the system by 2 VDC and verify the RPM recovers within 60 seconds | (success expected) |
| Reset the driving voltage and verify the RPM recovers within 60 seconds | (success expected) |

### Cleanup
After completing the firmware testing, ALWAYS reset the system to the defaults for operation. Adjust any custom
settings as desired.
