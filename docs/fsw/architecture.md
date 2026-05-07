# Flight Software Architecture

## Overview

The TEMPEST flight software (`main.py`) runs on a Raspberry Pi Zero and serves as the central command processor for the satellite. It receives commands via radio uplink, processes them through the appropriate subsystem, and returns binary telemetry packets via radio downlink.

## System Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           Raspberry Pi Zero              │
                    │                                          │
  Radio Uplink ───►│  main.py                                 │
  (RFM69/RFM95)    │    ├── Command Router                   │
                    │    ├── Beacon Thread                    │
  Radio Downlink◄──│    ├── Config Watchdog                  │
  (RFM69/RFM95)    │    │                                    │
                    │    ├── subsystems/                      │
                    │    │     ├── environmental.py ──► UART ──► RP2040 MCU
                    │    │     ├── obc.py (psutil)           │
                    │    │     ├── payload.py (rpicam)        │
                    │    │     ├── solar.py ──► I2C ──► INA219 x4
                    │    │     ├── eps.py ──► GPIO UART ──► ATmega328PB
                    │    │     └── retransmit.py              │
                    │    │                                    │
                    │    └── utils/                           │
                    │          ├── radio.py (SPI)             │
                    │          └── display.py (console)       │
                    └─────────────────────────────────────────┘
```

## Main Loop

The FSW operates in a continuous receive-process-respond loop:

1. **Receive**: Wait for a command string on the uplink radio
2. **Route**: Match the command string to a handler
3. **Process**: Execute the subsystem function
4. **Pack**: Encode the response as a binary telemetry packet using `struct.pack`
5. **Transmit**: Send the packet(s) on the downlink radio

```python
while True:
    packet = receive_uplink_message(uplink)
    if packet:
        command = packet.decode('utf-8').strip()
        response = process_command(command)
        send_downlink_message(downlink, response)
```

## Response Types

Commands produce one of several response types:

| Type | Description | Example |
|------|-------------|---------|
| **Single packet** | One binary struct packet | `GET_GYRO` → 16-byte GYRO packet |
| **Multi-packet list** | List of binary packets sent sequentially | `OBC_PROCESSES` → multiple OBCP chunks |
| **Windowed transfer** | Paced image transfer with delay between packets | `SEND_IMAGE` → hundreds of SEND packets at 0.25s intervals |
| **Fire-and-forget** | No response expected | `MORSE`, `RESET_ENV` |

## Health Beacon

A background thread broadcasts satellite health telemetry at a configurable interval (default 5 seconds). The beacon packet contains:

- Uptime (seconds since boot)
- CPU usage (%)
- RAM usage (%)
- Disk usage (%)
- Temperature (°C)

The beacon can be enabled/disabled via `BEACON_ON [interval]` and `BEACON_OFF` commands.

## Configuration Watchdog

The FSW monitors `config.yml` for changes using the `watchdog` library. When the file is modified, the radio configuration is reloaded without restarting the service. On startup, `config.yml.bak` is copied to `config.yml` to ensure a known-good configuration.

## CLI Development Mode

`cli_fsw.py` provides a command-line interface version of the FSW for development and testing. It uses `input()` for command entry instead of radio and returns raw data without `struct.pack` encoding. This allows testing command handlers without radio hardware.

## Startup Sequence

1. Restore `config.yml` from `config.yml.bak`
2. Parse configuration
3. Initialize radio(s) via `setup_radios()`
4. Initialize subsystem connections (serial, I2C, GPIO)
5. Perform throwaway `POLL` to flush stale UART data from RP2040
6. Start beacon thread (if configured)
7. Start config watchdog thread
8. Enter main receive loop
