# Telemetry Reference

## Packet Structure

All downlink telemetry packets share a common format:

```
[4-byte ASCII identifier][payload bytes][\n]
```

The `\n` newline is appended by `radio.py` during transmission and serves as a packet delimiter. It is not part of the logical payload.

## Binary Packing

Telemetry payloads are packed using Python's `struct.pack` with little-endian byte order:

| Format Code | Type | Size |
|-------------|------|------|
| `s` | char (ASCII string) | 1 byte each |
| `i` | signed int32 | 4 bytes |
| `I` | unsigned int32 | 4 bytes |
| `f` | float32 (IEEE 754) | 4 bytes |

## Telemetry Packet Definitions

### GYRO — Gyroscope

```
struct.pack('4s3f', b'GYRO', x, y, z)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | float | X | °/s |
| 8 | 4 | float | Y | °/s |
| 12 | 4 | float | Z | °/s |

**Total: 16 bytes**

### ACCL — Accelerometer

Same format as GYRO. Units: m/s².

### MAGN — Magnetometer

Same format as GYRO. Units: µT (microtesla).

### GRAV — Gravity Vector

Same format as GYRO. Units: m/s².

### EULR — Euler Angles

Same format as GYRO. Units: degrees (°).

### BME2 — BME280 Environmental

```
struct.pack('4s3f', b'BME2', temperature, pressure, altitude)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | float | Temperature | °C |
| 8 | 4 | float | Pressure | hPa |
| 12 | 4 | float | Altitude | m |

**Total: 16 bytes**

### TEMP — Temperature (IMU)

```
struct.pack('4si', b'TEMP', temperature)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | int32 | Temperature | °C |

**Total: 8 bytes**

### QUAT — Quaternion

```
struct.pack('4s4f', b'QUAT', w, x, y, z)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | float | W | — |
| 8 | 4 | float | X | — |
| 12 | 4 | float | Y | — |
| 16 | 4 | float | Z | — |

**Total: 20 bytes**

### ADCS — Attitude Determination

```
struct.pack('4s7f', b'ADCS', heading, roll, pitch, quat_w, quat_x, quat_y, quat_z)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | float | Heading | ° |
| 8 | 4 | float | Roll | ° |
| 12 | 4 | float | Pitch | ° |
| 16 | 4 | float | Quat W | — |
| 20 | 4 | float | Quat X | — |
| 24 | 4 | float | Quat Y | — |
| 28 | 4 | float | Quat Z | — |

**Total: 32 bytes**

### SOLR — Solar Panels

```
struct.pack('4s8f', b'SOLR', v1, i1, v2, i2, v3, i3, v4, i4)
```

Panel order matches INA219 addresses:

| Index | Address | Panel |
|-------|---------|-------|
| 0 | 0x40 | X- |
| 1 | 0x41 | X+ |
| 2 | 0x44 | Y+ |
| 3 | 0x45 | Y- |

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | float | Panel 1 Voltage | V |
| 8 | 4 | float | Panel 1 Current | mA |
| 12 | 4 | float | Panel 2 Voltage | V |
| 16 | 4 | float | Panel 2 Current | mA |
| 20 | 4 | float | Panel 3 Voltage | V |
| 24 | 4 | float | Panel 3 Current | mA |
| 28 | 4 | float | Panel 4 Voltage | V |
| 32 | 4 | float | Panel 4 Current | mA |

**Total: 36 bytes**

### EPSS — EPS Status

```
struct.pack('4s5if', b'EPSS', error, ch1, ch2, ch3, ch4, battery_voltage)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | int32 | Error Code | — |
| 8 | 4 | int32 | Channel 1 | 0=OFF, 1=ON |
| 12 | 4 | int32 | Channel 2 | 0=OFF, 1=ON |
| 16 | 4 | int32 | Channel 3 | 0=OFF, 1=ON |
| 20 | 4 | int32 | Channel 4 | 0=OFF, 1=ON |
| 24 | 4 | float | Battery Voltage | V |

**Total: 28 bytes**

### OBCC / OBCR / OBCD — OBC Single Values

```
struct.pack('4sf', b'OBCC', value)  # CPU, RAM, or Disk
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | float | Usage | % |

**Total: 8 bytes**

### HOST — Hostname

```
struct.pack('4s11s', b'HOST', hostname_bytes)
```

| Offset | Size | Type | Field |
|--------|------|------|-------|
| 0 | 4 | char[4] | Identifier |
| 4 | 11 | char[11] | Hostname (null-padded) |

**Total: 15 bytes**

### BECN — Health Beacon

```
struct.pack('4sIffff', b'BECN', uptime, cpu, ram, disk, temp)
```

| Offset | Size | Type | Field | Unit |
|--------|------|------|-------|------|
| 0 | 4 | char[4] | Identifier | — |
| 4 | 4 | uint32 | Uptime | seconds |
| 8 | 4 | float | CPU Usage | % |
| 12 | 4 | float | RAM Usage | % |
| 16 | 4 | float | Disk Usage | % |
| 20 | 4 | float | Temperature | °C |

**Total: 24 bytes**

### XFRC — Transfer Complete

```
struct.pack('4sI', b'XFRC', total_packets)
```

| Offset | Size | Type | Field |
|--------|------|------|-------|
| 0 | 4 | char[4] | Identifier |
| 4 | 4 | uint32 | Total Packets |

**Total: 8 bytes**

### OBCP / OBCL — Chunked Text Data

These are variable-length, multi-packet responses. Each chunk:

```
[4-byte ID][text data up to (chunk_size - 5) bytes]
```

The ground station accumulates chunks and flushes after 1 second of no new data.

### SEND — Image Data Packet

```
SEND[base64_data][total_chunks:04d][current_chunk:04d]
```

Image packets are text-based (not binary struct). See [Image Transfer](../ground-station/image-transfer.md).

### PHOT — Photo Response

```
struct.pack('4sI', b'PHOT', filename_length) + filename_bytes
```

Variable length. Contains the filename of the captured image.
