# Command Protocol

## Overview

Commands are sent as plain text strings via radio uplink, terminated with a newline. The FSW parses the command string, executes the appropriate handler, and returns one or more binary telemetry packets via downlink.

## Command Format

```
COMMAND_NAME [arg1] [arg2] ...
```

Commands are case-sensitive and uppercase. Arguments are space-separated.

## Environmental Sensor Commands

These commands communicate with the RP2040 sensor MCU via UART at 9600 baud.

| Command | Response ID | Description |
|---------|-------------|-------------|
| `ENV_POLL` | `POLL` | Poll all environmental sensors (17 float values) |
| `GET_GYRO` | `GYRO` | Gyroscope X, Y, Z (°/s) |
| `GET_ACCEL` | `ACCL` | Accelerometer X, Y, Z (m/s²) |
| `GET_MAG` | `MAGN` | Magnetometer X, Y, Z (µT) |
| `GET_GRAVITY` | `GRAV` | Gravity vector X, Y, Z (m/s²) |
| `GET_EULER` | `EULR` | Euler angles X, Y, Z (°) |
| `GET_QUATERNION` | `QUAT` | Quaternion W, X, Y, Z |
| `GET_ORIENTATION` | `ADCS` | Heading, Roll, Pitch + Quaternion W, X, Y, Z |
| `GET_BME` | `BME2` | Temperature (°C), Pressure (hPa), Altitude (m) |
| `GET_TEMP` | `TEMP` | Temperature from BNO055 IMU (°C, integer) |
| `MORSE <message>` | — | Play morse code on buzzer (fire-and-forget) |
| `RESET_ENV` | — | Reset the RP2040 MCU (fire-and-forget) |

## Onboard Computer Commands

| Command | Response ID | Description |
|---------|-------------|-------------|
| `OBC_CPU` | `OBCC` | CPU usage (%) |
| `OBC_RAM` | `OBCR` | RAM usage (%) |
| `OBC_DISK` | `OBCD` | Disk usage (%) |
| `OBC_PROCESSES` | `OBCP` | Running process list (multi-packet, chunked) |
| `OBC_LIST_FILES [dir]` | `OBCL` | File listing (multi-packet, chunked). Defaults to `capture` directory |
| `GET_HOSTNAME` | `HOST` | Satellite hostname (11-byte string) |
| `WIFI_INFO` | `WIFI` | WiFi SSID and IP address of the satellite AP |
| `OBC_RESTART` | — | Plays morse "A" then reboots the Pi |
| `OBC_SHUTDOWN` | — | Shuts down the Pi |

## Power System Commands

| Command | Response ID | Description |
|---------|-------------|-------------|
| `GET_SOLAR` | `SOLR` | 4 solar panels: voltage (V) + current (mA) each |
| `EPS_STATUS` | `EPSS` | EPS error code, 4 channel states, battery voltage |
| `EPS CH<n>,<0\|1>` | `EPSS` | Set EPS channel n on (1) or off (0) |

## Payload Commands

| Command | Response ID | Description |
|---------|-------------|-------------|
| `TAKE_PHOTO` | `PHOT` | Capture image, returns filename |
| `SEND_IMAGE <file> [chunk_sz]` | `SEND` | Transmit image file (paced transfer) |
| `RETRANSMIT <file> <pkt1> [pkt2...]` | `RETX` | Retransmit specific packets |

## Beacon Commands

| Command | Response ID | Description |
|---------|-------------|-------------|
| `BEACON_ON [interval]` | `BECN` | Enable health beacon (default 5s interval) |
| `BEACON_OFF` | `BECN` | Disable health beacon |

## Multi-Packet Responses

Some commands return data too large for a single radio packet. These use the `chunk_data()` function to split the response:

```python
def chunk_data(data, identifier, max_chunk_size=None):
    # max_chunk_size defaults to (radio_chunk_size - 5)
    # 4 bytes for identifier + 1 byte for \n appended by radio.py
    # Each chunk: [4-byte ID][payload_data]
```

Multi-packet responses are sent with a delay between packets (`DEFAULT_TX_DELAY = 0.25s`) to avoid overwhelming the receiver.

## Paced Image Transfer

Image transfer uses a special windowed mechanism:

1. FSW returns `('WINDOWED', [packet_list])` tuple
2. Main loop calls `send_paced()` which transmits each packet with a 0.25s delay
3. Progress is logged every 50 packets
4. After all packets: an `XFRC` packet is sent with the total packet count
5. Ground station checks for missing packets and can request retransmission

See [Image Transfer](../ground-station/image-transfer.md) for the full transfer protocol.
