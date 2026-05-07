# Sensor MCU (RP2040)

## Overview

The RP2040 Pico runs CircuitPython firmware (`code.py`) and serves as the environmental sensor interface. It communicates with the Pi Zero FSW over UART and manages:

- BNO055 9-DOF IMU (accelerometer, gyroscope, magnetometer)
- BMP280 barometric pressure / temperature / altitude sensor
- Piezo buzzer for morse code output

![Placeholder: RP2040 Pico with BNO055 and BMP280 connected](../img/placeholder-rp2040-sensors.png)

## Hardware Connections

### I2C Bus

| Pin | Function |
|-----|----------|
| GP27 | SCL |
| GP26 | SDA |

**I2C Devices:**

| Device | Address | Function |
|--------|---------|----------|
| BNO055 | 0x28 (default) | 9-DOF IMU |
| BMP280 | 0x76 | Pressure, temperature, altitude |

### UART

| Pin | Function |
|-----|----------|
| GP0 | RX (from Pi Zero TX) |
| GP1 | TX (to Pi Zero RX) |

**UART Parameters:** 9600 baud, 8N1

### Buzzer

| Pin | Function |
|-----|----------|
| GP15 | Digital output to piezo buzzer |

## Command Protocol

The RP2040 receives commands as uppercase ASCII strings terminated with `\n`. Responses are Python list literals terminated with `\n`.

### Sensor Commands

| Command | Response Format | Description |
|---------|-----------------|-------------|
| `GET_GYRO` | `[x, y, z]` | Gyroscope (°/s) |
| `GET_ACCEL` | `[x, y, z]` | Accelerometer (m/s²) |
| `GET_MAG` | `[x, y, z]` | Magnetometer (µT) |
| `GET_GRAVITY` | `[x, y, z]` | Gravity vector (m/s²) |
| `GET_EULER` | `[yaw, roll, pitch]` | Euler angles (°) |
| `GET_QUATERNION` | `[w, x, y, z]` | Quaternion orientation |
| `GET_BME` | `[temp, pressure, altitude]` | BME280 readings |
| `GET_TEMP` | `[temp]` | BNO055 die temperature (°C) |
| `POLL` | `[16 float values]` | All sensor data concatenated |

### System Commands

| Command | Response | Description |
|---------|----------|-------------|
| `MORSE <message>` | `OK` | Play morse code on buzzer |
| `RESET` | — | Soft reset MCU (`microcontroller.reset()`) |

## Response Parsing

The Pi Zero FSW parses RP2040 responses using `ast.literal_eval()`:

```python
# RP2040 sends: [1.23, -0.45, 0.78]\n
# Pi Zero receives and parses:
response = ast.literal_eval("[1.23, -0.45, 0.78]")
# result: [1.23, -0.45, 0.78] (Python list of floats)
```

This approach safely evaluates Python literals without executing arbitrary code.

## Morse Code

The buzzer on GP15 supports morse code playback with the following timing:

| Element | Duration |
|---------|----------|
| Dot | 0.1s |
| Dash | 0.3s |
| Symbol space | 0.1s |
| Letter space | 0.3s |
| Word space | 0.7s |

On boot, the MCU plays "Tempest" in morse code as a startup indicator.

## BNO055 Sensor Modes

The BNO055 provides multiple data types from its integrated sensors:

| Data Type | Source | Values |
|-----------|--------|--------|
| Gyroscope | Angular rate sensor | X, Y, Z rotation rates |
| Accelerometer | Linear acceleration sensor | X, Y, Z acceleration |
| Magnetometer | Geomagnetic sensor | X, Y, Z field strength |
| Gravity | Sensor fusion | X, Y, Z gravity components |
| Euler | Sensor fusion | Heading, Roll, Pitch |
| Quaternion | Sensor fusion | W, X, Y, Z |
| Temperature | On-die sensor | Integer °C |

## First-Read Behavior

The first sensor read after MCU startup may return stale data from the UART buffer. The FSW performs a throwaway `POLL` command during initialization to flush this stale data. Subsequent reads return current values.

![Placeholder: RP2040 wiring diagram](../img/placeholder-rp2040-wiring.png)
