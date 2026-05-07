# Radio Configuration

## Overview

The ground station can configure both the uplink and downlink radio parameters through the browser interface. Configuration commands are sent to the GS Pico MCU, which applies them to its radio hardware.

## Configuration Panel

![Placeholder: Radio configuration panel in browser](../img/placeholder-gs-radio-panel.png)

### Uplink Configuration (Ground to Satellite)

| Parameter | Options | Default |
|-----------|---------|---------|
| Radio Type | RFM95 (LoRa), RFM69 (FSK) | RFM95 |
| Band | 433 MHz, 868 MHz, 915 MHz | 915 MHz |
| Frequency | Within band range | 915.0 MHz |
| Satellite Node | 0-255 | 255 (broadcast) |

### Downlink Configuration (Satellite to Ground)

| Parameter | Options | Default |
|-----------|---------|---------|
| Radio Type | RFM95 (LoRa), RFM69 (FSK) | RFM95 |
| Band | 433 MHz, 868 MHz, 915 MHz | 915 MHz |
| Frequency | Within band range | 915.0 MHz |
| Ground Node | 0-255 | 255 (broadcast) |

## ISM Band Frequency Ranges

| Band | Min (MHz) | Max (MHz) | Region | Default |
|------|-----------|-----------|--------|---------|
| 433 | 433.05 | 434.79 | Global | 433.92 |
| 868 | 863.0 | 870.0 | EU | 869.5 |
| 915 | 902.0 | 928.0 | US | 915.0 |

The frequency input validates against the selected band range. Out-of-range values are highlighted with a red border.

## Applying Configuration

Click **"Apply Configuration"** to send both uplink and downlink settings:

```
set_uplink_radio rfm69 915.0 255
set_downlink_radio rfm69 915.0 255
```

There is a 1-second delay between the uplink and downlink configuration commands.

## Querying Current Configuration

Click **"Query Current Config"** to request the current radio settings from the GS Pico MCU:

```
get_radio_config
```

The response updates the UI fields with the current hardware configuration.

## LED Indicators

Four LED indicators show radio activity:

| LED | Meaning |
|-----|---------|
| RFM95 TX | LoRa transmission (uplink) |
| RFM95 RX | LoRa reception (downlink) |
| RFM69 TX | FSK transmission (uplink) |
| RFM69 RX | FSK reception (downlink) |

The active LED pair depends on the configured radio types. LEDs flash briefly (100ms) on each packet.

## Radio Characteristics

### RFM69 (FSK)

| Parameter | Value |
|-----------|-------|
| Modulation | FSK/OOK |
| FIFO Size | 66 bytes |
| Library Header | 4 bytes |
| Max Payload | ~50 bytes |
| Data Rate | Higher |
| Range | Shorter |

### RFM95 (LoRa)

| Parameter | Value |
|-----------|-------|
| Modulation | LoRa (CSS) |
| FIFO Size | 252 bytes |
| Library Header | 4 bytes |
| Max Payload | ~240 bytes |
| Data Rate | Lower |
| Range | Longer |

## Matching Satellite Configuration

The ground station radio settings must match the satellite's `config.yml`:

| GS Setting | Satellite Config |
|------------|-----------------|
| Uplink Radio Type | `Tempest.Comms.uplink.radio` |
| Uplink Frequency | `Tempest.Comms.uplink.freq` |
| Downlink Radio Type | `Tempest.Comms.downlink.radio` |
| Downlink Frequency | `Tempest.Comms.downlink.freq` |

If the radio types or frequencies don't match, communication will fail silently — no error is reported.

## Encryption

Both the satellite and ground station support AES encryption (configured in `config.yml`). When enabled, both ends must use the same 16-byte encryption key. Encryption configuration is currently only available through `config.yml` on the satellite side, not through the browser interface.
