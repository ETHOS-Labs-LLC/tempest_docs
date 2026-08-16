# Telemetry ID Quick Reference

## All Telemetry Identifiers

| ID | Name | Size (bytes) | Format | Source Command |
|----|------|-------------|--------|----------------|
| `POLL` | Environmental Poll | Variable | `4s + I + Nf` | `ENV_POLL` |
| `ADCS` | Attitude Determination | 32 | `4s + 7f` | `GET_ORIENTATION` |
| `BME2` | BME280 Environment | 16 | `4s + 3f` | `GET_BME` |
| `GYRO` | Gyroscope | 16 | `4s + 3f` | `GET_GYRO` |
| `ACCL` | Accelerometer | 16 | `4s + 3f` | `GET_ACCEL` |
| `MAGN` | Magnetometer | 16 | `4s + 3f` | `GET_MAG` |
| `TEMP` | Temperature | 8 | `4s + i` | `GET_TEMP` |
| `GRAV` | Gravity Vector | 16 | `4s + 3f` | `GET_GRAVITY` |
| `EULR` | Euler Angles | 16 | `4s + 3f` | `GET_EULER` |
| `QUAT` | Quaternion | 20 | `4s + 4f` | `GET_QUATERNION` |
| `SOLR` | Solar Panels | 36 | `4s + 8f` | `GET_SOLAR` |
| `HOST` | Hostname | 15 | `4s + 11s` | `GET_HOSTNAME` |
| `WIFI` | WiFi SSID + IP | 51 | `4s + 32s + 15s` | `WIFI_INFO` |
| `PHOT` | Photo Captured | Variable | `4s + I + data` | `TAKE_PHOTO` |
| `OBCP` | Process List | Variable | `4s + text` | `OBC_PROCESSES` |
| `OBCC` | CPU Usage | 8 | `4s + f` | `OBC_CPU` |
| `OBCR` | RAM Usage | 8 | `4s + f` | `OBC_RAM` |
| `OBCD` | Disk Usage | 8 | `4s + f` | `OBC_DISK` |
| `OBCL` | File Listing | Variable | `4s + text` | `OBC_LIST_FILES` |
| `SEND` | Image Data | Variable | Text | `SEND_IMAGE` |
| `RETX` | Retransmit Data | Variable | Text | `RETRANSMIT` |
| `EPSS` | EPS Status | 28 | `4s + 5i + f` | `EPS_STATUS` |
| `BECN` | Health Beacon | 24 | `4s + I + 4f` | `BEACON_ON` |
| `XFRC` | Transfer Complete | 8 | `4s + I` | (auto after SEND_IMAGE) |

## Response Categories

### Fixed-Size Binary (struct.pack)

These packets always have the same size and are parsed using known byte offsets:

```
GYRO, ACCL, MAGN, GRAV, EULR  →  16 bytes (3 floats)
BME2                            →  16 bytes (3 floats)
TEMP                            →   8 bytes (1 int)
QUAT                            →  20 bytes (4 floats)
ADCS                            →  32 bytes (7 floats)
SOLR                            →  36 bytes (8 floats)
EPSS                            →  28 bytes (5 ints + 1 float)
OBCC, OBCR, OBCD               →   8 bytes (1 float)
HOST                            →  15 bytes (11-char string)
WIFI                            →  51 bytes (32-char SSID + 15-char IP)
BECN                            →  24 bytes (1 uint + 4 floats)
XFRC                            →   8 bytes (1 uint)
```

### Multi-Packet Chunked (newline-delimited)

These are split across multiple radio packets. The ground station accumulates chunks and flushes after a 1-second timeout:

```
OBCP  →  Process list (chunked text)
OBCL  →  File listing (chunked text)
```

### Image Transfer (newline-delimited)

```
SEND  →  Base64 image chunk + 8-digit metadata
RETX  →  Retransmitted image chunk
```

### Variable-Length Binary

```
POLL  →  Length-prefixed sensor data
PHOT  →  Length-prefixed filename
```

## Struct Format Codes

| Code | Python Type | Size | JavaScript Equivalent |
|------|-------------|------|-----------------------|
| `s` | char | 1 byte | `TextDecoder` |
| `i` | int32 | 4 bytes | `DataView.getInt32(offset, true)` |
| `I` | uint32 | 4 bytes | `DataView.getUint32(offset, true)` |
| `f` | float32 | 4 bytes | `DataView.getFloat32(offset, true)` |

All multi-byte values use **little-endian** byte order (the `true` parameter in JavaScript DataView methods).
