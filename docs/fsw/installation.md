# Installation

## Prerequisites

- Raspberry Pi Zero W (or Pi Zero 2 W)
- Raspbian / Raspberry Pi OS Lite
- Python 3.x
- Hardware interfaces enabled: I2C, SPI, UART

## Quick Install

Run the install script from the project root:

```bash
sudo bash install.sh
```

This script performs the following:

### 1. System Update

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Python Dependencies

| Package | Purpose |
|---------|---------|
| `pyyaml` | Configuration file parsing |
| `psutil` | System metrics (CPU, RAM, disk) |
| `pyserial` | UART communication with RP2040 |
| `adafruit-circuitpython-rfm9x` | RFM95 LoRa radio driver |
| `adafruit-circuitpython-rfm69` | RFM69 FSK radio driver |
| `adafruit-extended-bus` | Extended SPI/I2C bus support |
| `adafruit-ina219` | INA219 current sensor driver |
| `watchdog` | File system monitoring for config reload |
| `pigpio` | Software UART for EPS communication |

### 3. Hardware Interface Configuration

The script enables the following via `raspi-config`:

- **I2C** — For INA219 solar sensors
- **SPI** — For RFM69/RFM95 radio modules
- **UART** — For RP2040 MCU communication

### 4. pigpiod Service

The `pigpio` daemon is required for bit-banged UART to the ATmega328PB EPS:

```bash
sudo systemctl enable pigpiod
sudo systemctl start pigpiod
```

### 5. Systemd Service

The FSW runs as a systemd service that starts on boot and restarts on failure:

```ini
[Unit]
Description=TEMPEST Flight Software
After=multi-user.target pigpiod.service

[Service]
ExecStart=/usr/bin/python /home/ethos/TEMPEST_FSW/main.py
Restart=always
User=ethos

[Install]
WantedBy=multi-user.target
```

## Service Management

```bash
# Start the FSW
sudo systemctl start tempest_fsw

# Stop the FSW
sudo systemctl stop tempest_fsw

# Check status
sudo systemctl status tempest_fsw

# View logs
journalctl -u tempest_fsw -f

# Restart after code changes
sudo systemctl restart tempest_fsw
```

## Manual Execution

For development and debugging, run the FSW directly:

```bash
# Production mode (requires radio hardware)
python main.py

# CLI development mode (no radio required)
python cli_fsw.py
```

## RP2040 Firmware

The RP2040 Pico runs CircuitPython. To update the firmware:

1. Hold the BOOTSEL button while connecting the Pico via USB
2. Copy the CircuitPython `.uf2` file to the mounted drive
3. After reboot, copy `code.py` to the Pico's `CIRCUITPY` drive
4. Install required CircuitPython libraries:
    - `adafruit_bno055`
    - `adafruit_bmp280`

See [Sensor MCU](sensor-mcu.md) for details on the RP2040 firmware.

## Directory Structure

```
TEMPEST_FSW/
├── main.py              # Production FSW (radio mode)
├── cli_fsw.py           # Development FSW (CLI mode)
├── code.py              # RP2040 CircuitPython firmware
├── config.yml           # Radio configuration (live-reloaded)
├── config.yml.bak       # Backup config (restored on startup)
├── install.sh           # Installation script
├── fsw_service.sh       # Systemd service creator
├── capture/             # Image capture directory
├── subsystems/
│   ├── environmental.py # RP2040 UART interface
│   ├── obc.py           # On-board computer functions
│   ├── payload.py       # Camera and image handling
│   ├── solar.py         # INA219 solar panel sensors
│   ├── eps.py           # ATmega328PB EPS interface
│   └── retransmit.py    # Packet retransmission
├── utils/
│   ├── radio.py         # Radio abstraction layer
│   └── display.py       # Console output formatting
├── FSC-GS/              # Ground station web interface
│   ├── index.html
│   ├── styles.css
│   └── scripts/
│       ├── main.js
│       ├── telemetry.js
│       ├── serial-communication.js
│       ├── image-processing.js
│       └── 3d-visualization.js
└── docs/                # This documentation
```

![Placeholder: Assembled TEMPEST flat-sat hardware](../img/placeholder-assembled-hardware.png)
