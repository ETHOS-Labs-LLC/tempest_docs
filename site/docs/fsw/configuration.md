# Configuration

## config.yml

The FSW reads its configuration from `config.yml` in the project root. This file controls radio parameters and is monitored for live changes via the `watchdog` library.

### Structure

```yaml
Tempest:
  Comms:
    uplink:
      radio: rfm69           # rfm69 or rfm95
      freq: 915              # Frequency in MHz
      encryption: false
      enc_key: ""

    downlink:
      radio: rfm69           # rfm69 or rfm95
      freq: 915              # Frequency in MHz
      chunk_sizes:
        rfm95: 240            # Max payload bytes for RFM95
        rfm69: 50             # Max payload bytes for RFM69
      encryption: false
      enc_key: ""

    crosslink:
      radio: "none"           # Future feature
      frequency: 921
      encryption: false
      enc_key: ""
```

### Configuration Parameters

#### Uplink (Ground to Satellite)

| Parameter | Type | Description |
|-----------|------|-------------|
| `radio` | string | Radio module type: `rfm69` or `rfm95` |
| `freq` | number | Operating frequency in MHz |
| `encryption` | boolean | Enable AES encryption |
| `enc_key` | string | 16-byte AES encryption key |

#### Downlink (Satellite to Ground)

| Parameter | Type | Description |
|-----------|------|-------------|
| `radio` | string | Radio module type: `rfm69` or `rfm95` |
| `freq` | number | Operating frequency in MHz |
| `chunk_sizes.rfm95` | number | Max payload size for RFM95 (default 240) |
| `chunk_sizes.rfm69` | number | Max payload size for RFM69 (default 50) |
| `encryption` | boolean | Enable AES encryption |
| `enc_key` | string | 16-byte AES encryption key |

### Live Reload

The FSW uses `watchdog.observers.Observer` with a `FileSystemEventHandler` to detect changes to `config.yml`. When modified:

1. The watchdog triggers a reload callback
2. New configuration is parsed from YAML
3. Radio parameters are updated without restarting the service

### Backup Restoration

On startup, `config.yml.bak` is copied over `config.yml` to ensure a known-good configuration. This prevents a corrupted config from bricking the satellite after a reboot.

```
Startup: config.yml.bak → config.yml → parse → initialize radios
```

## Radio Hardware Configuration

### SPI Bus Assignments

The FSW uses two SPI buses for dual-radio operation:

**Primary SPI (Hardware):**

| Signal | GPIO |
|--------|------|
| SCLK | 21 |
| MOSI | 20 |
| MISO | 19 |
| CS | 5 |
| RESET | 17 |

**Secondary SPI (Bitbanged):**

| Signal | GPIO |
|--------|------|
| SCLK | 21 |
| MOSI | 20 |
| MISO | 19 |
| CS | 6 |
| RESET | 26 |

### Radio Sharing

If both uplink and downlink are configured to use the same radio type (e.g., both `rfm69`), they share a single radio instance. This means communication is half-duplex — the satellite cannot transmit and receive simultaneously.

### ISM Band Frequency Ranges

| Band | Min (MHz) | Max (MHz) | Region |
|------|-----------|-----------|--------|
| 433 | 433.05 | 434.79 | Global |
| 868 | 863.0 | 870.0 | EU |
| 915 | 902.0 | 928.0 | US |

### Chunk Size Limits

These limits account for the radio FIFO size and the Adafruit library header:

| Radio | FIFO Size | Library Header | Usable Payload | Config Default |
|-------|-----------|----------------|----------------|----------------|
| RFM69 | 66 bytes | 4 bytes | 62 bytes | 50 bytes |
| RFM95 | 252 bytes | 4 bytes | 248 bytes | 240 bytes |

The `\n` delimiter appended by `radio.py` consumes 1 additional byte from the usable payload.

## Node Addressing

Radio node addresses identify the sender and destination:

- **Default:** 0xFF (255) broadcast
- **FSW Node ID:** Derived from hostname: `int(hostname[-2:]) * 10`
- **Ground Station Node:** Configurable via the ground station interface

![Placeholder: Radio module SPI wiring on Pi Zero](../img/placeholder-radio-wiring.png)
