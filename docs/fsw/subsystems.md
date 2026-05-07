# Subsystems

## Overview

The FSW interfaces with satellite hardware through dedicated subsystem modules in the `subsystems/` directory. Each module encapsulates communication with a specific hardware component.

```
subsystems/
├── environmental.py   # RP2040 sensor MCU (UART)
├── obc.py             # On-board computer functions (psutil)
├── payload.py         # Camera capture and image handling
├── solar.py           # INA219 solar panel sensors (I2C)
├── eps.py             # ATmega328PB EPS (bit-bang UART)
└── retransmit.py      # Packet retransmission from .chunked files
```

## Environmental Subsystem

**File:** `subsystems/environmental.py`

Manages communication with the RP2040 sensor MCU via hardware UART.

**Connection Parameters:**

| Parameter | Value |
|-----------|-------|
| Port | `/dev/serial0` |
| Baud Rate | 9600 |
| Timeout | 1 second |

**Class: `SerialConnection`**

The `send_command()` method handles all RP2040 communication:

1. Flushes input/output buffers
2. Sends command as uppercase string + `\n`
3. Waits 50ms for processing
4. Reads response with 2-second timeout
5. Parses response using `ast.literal_eval()`
6. Retries up to 3 times on failure

Fire-and-forget commands (`MORSE`, `RESET`) skip the response wait.

**Data Flow:**

```
Pi Zero                          RP2040
  │                                │
  ├──── "GET_GYRO\n" ────────────►│
  │                                ├── Read BNO055 sensor
  │◄──── "[1.23, -0.45, 0.78]\n" ─┤
  │                                │
  ├──── ast.literal_eval() ───►  [1.23, -0.45, 0.78]
```

![Placeholder: UART wiring between Pi Zero and RP2040](../img/placeholder-uart-wiring.png)

## On-Board Computer (OBC)

**File:** `subsystems/obc.py`

Provides system information using `psutil` and manages image file operations.

**Functions:**

| Function | Returns | Description |
|----------|---------|-------------|
| `get_processes()` | string | Comma-separated list of running process names |
| `get_cpu_usage()` | float | CPU utilization percentage |
| `get_ram_usage()` | float | RAM utilization percentage |
| `get_disk_usage()` | float | Disk utilization percentage |
| `get_hostname()` | bytes | Hostname padded/truncated to 11 bytes |
| `list_files(dir)` | string | Comma-separated file listing. Defaults to `capture` |
| `reboot()` | — | Executes `sudo reboot` |

**Image Processing Pipeline:**

```
send_image(filename, chunk_size)
  └── process_all_chunks(filename, chunk_size)
        ├── base64_encode_chunks()  ← Generator: reads raw file in chunks
        ├── Format: [base64_data][total:04d][current:04d]
        ├── Writes .chunked file for retransmission
        └── Returns list of packet strings
```

**Chunk Sizes by Radio:**

| Radio | Raw Chunk | Base64 | + Metadata | Total Payload |
|-------|-----------|--------|------------|---------------|
| RFM69 | 33 bytes | 44 chars | + 8 digits | ~52 bytes |
| RFM95 | 168 bytes | 224 chars | + 8 digits | ~232 bytes |

## Payload Camera

**File:** `subsystems/payload.py`

**Class: `PayloadCamera`**

Captures photos using the Raspberry Pi camera module and prepares them for transmission.

**Capture Process:**

1. Execute `rpicam-still -q 40 -o capture/{count}.jpg`
2. Gzip compress the JPEG
3. Create `.chunked` file for transmission

![Placeholder: Sample captured image in ground station](../img/placeholder-image-capture.png)

## Solar Panel Monitoring

**File:** `subsystems/solar.py`

Reads voltage and current from four INA219 current sensors on the I2C bus.

**Sensor Mapping:**

| I2C Address | Panel | Physical Location |
|-------------|-------|-------------------|
| 0x40 | X- | Negative X face |
| 0x41 | X+ | Positive X face |
| 0x44 | Y+ | Positive Y face |
| 0x45 | Y- | Negative Y face |

**Configuration:** 12-bit ADC resolution, 16V bus voltage range.

**Return Format:** 8 float values — `[V1, I1, V2, I2, V3, I3, V4, I4]`

If a sensor is unavailable, it returns `0.000` for both voltage and current.

![Placeholder: Solar panel INA219 sensor wiring](../img/placeholder-solar-wiring.png)

## Electrical Power System (EPS)

**File:** `subsystems/eps.py`

Communicates with the ATmega328PB EPS controller via bit-banged UART using `pigpio`.

**Connection Parameters:**

| Parameter | Value |
|-----------|-------|
| TX GPIO | 27 |
| RX GPIO | 22 |
| Baud Rate | 9600 |
| Library | pigpio (software UART) |

**Protocol:**

Commands are sent as `&BMS [COMMAND]\n`. Responses are comma-separated values parsed as `[int, int, int, int, int, float]` representing error code, 4 channel states, and battery voltage.

```
Pi Zero                        ATmega328PB
  │                                │
  ├── "&BMS STATUS\n" ────────────►│
  │◄── "0,1,1,0,1,3.72\n" ────────┤
  │                                │
  └── Parse: error=0, ch1=ON, ch2=ON, ch3=OFF, ch4=ON, batt=3.72V
```

**Channel Control:**

```
EPS CH1,1    → Turn channel 1 ON
EPS CH3,0    → Turn channel 3 OFF
```

## Retransmission

**File:** `subsystems/retransmit.py`

Reads specific line numbers from `.chunked` files to support packet retransmission. When the ground station detects missing packets after an image transfer, it sends a `RETRANSMIT` command with the filename and packet numbers.

```
RETRANSMIT 5.jpg.gz.chunked 3 7 15 22
```

The retransmit module reads lines 3, 7, 15, and 22 from the `.chunked` file and returns them as individual packets.
