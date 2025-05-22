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
