# Packet Parsing

## Overview

The ground station browser receives raw bytes from the GS Pico MCU via USB CDC serial. Since USB CDC does not preserve packet boundaries, the browser must buffer incoming data and parse packet boundaries itself.

## Data Flow

```
Satellite Radio TX → GS Pico MCU Radio RX → USB CDC → Browser WebSerial Reader
                                                            │
                                                    ┌───────▼────────┐
                                                    │  dataBuffer    │
                                                    │  (Uint8Array)  │
                                                    └───────┬────────┘
                                                            │
                                                    ┌───────▼────────┐
                                                    │ Packet Router  │
                                                    │ (identifier)   │
                                                    └───────┬────────┘
                                                            │
                                    ┌───────────────┬───────┼───────────────┐
                                    ▼               ▼       ▼               ▼
                              Binary Packet    SEND/RETX  OBCP/OBCL    Text Line
                              (fixed size)    (newline)   (newline)    (newline)
```

## Buffering Strategy

All incoming bytes are appended to a `dataBuffer` (Uint8Array). The parser then attempts to extract complete packets from the front of the buffer:

```javascript
// Append new data
const newBuffer = new Uint8Array(dataBuffer.length + value.length);
newBuffer.set(dataBuffer);
newBuffer.set(value, dataBuffer.length);
dataBuffer = newBuffer;

// Parse loop - extract packets until buffer is exhausted
while (dataBuffer.length >= 4) {
    const identifier = decode first 4 bytes;
    // Route based on identifier...
}
```

## Packet Types and Parsing

### Fixed-Size Binary Packets

Known identifiers with fixed sizes are extracted directly:

```javascript
const packetSizes = {
    'GYRO': 16, 'ACCL': 16, 'MAGN': 16, 'GRAV': 16, 'EULR': 16,
    'BME2': 16, 'TEMP': 8,  'QUAT': 20,
    'OBCR': 8,  'OBCD': 8,  'OBCC': 8,
    'ADCS': 32, 'EPSS': 28, 'HOST': 15, 'WIFI': 51, 'SOLR': 36,
    'XFRC': 8,  'BECN': 24
};
```

If the buffer doesn't contain enough bytes for the expected packet size, parsing pauses until more data arrives.

### Newline-Delimited Packets

`SEND`, `RETX`, `OBCP`, and `OBCL` packets use newline (`\n`) as the delimiter. The parser scans for `0x0A` or `0x0D` after the 4-byte identifier:

```javascript
if (identifier === 'SEND' || identifier === 'RETX' ||
    identifier === 'OBCP' || identifier === 'OBCL') {
    // Find newline position
    let newlinePos = -1;
    for (let i = 4; i < dataBuffer.length; i++) {
        if (dataBuffer[i] === 0x0A || dataBuffer[i] === 0x0D) {
            newlinePos = i;
            break;
        }
    }
    if (newlinePos > 4) {
        const packet = dataBuffer.slice(0, newlinePos);
        // Route to appropriate handler...
        // Advance buffer past newline(s)
    } else {
        break; // Wait for more data
    }
}
```

This approach correctly handles USB CDC fragmentation — if a packet arrives split across multiple USB reads, the parser waits until the newline delimiter arrives before processing.

### Variable-Length Binary Packets

`POLL` and `PHOT` packets have variable sizes indicated by a length field in the packet header. They are handled as special cases in the parser.

### Text Lines

Unrecognized data (no matching identifier) is treated as a text line, parsed until a newline, and logged to the event log.

## Radio Configuration Parsing

Configuration responses from the GS Pico MCU use a special buffering mode. When a `CONFIG:` prefix is detected, the browser accumulates lines until parsing is complete, then updates the radio configuration UI.

## LED Blinking

On each received packet, the browser blinks the appropriate LED indicator and flashes the GSB downlink monitor:

```javascript
blinkLED('rfm95RxLed', 100);  // Also flashes monDownlink
```

On each sent command:

```javascript
blinkLED('rfm95TxLed', 100);  // Also flashes monUplink
```

The LED type (rfm95 vs rfm69) is determined by the current radio configuration.

## Error Handling

- **Incomplete packets**: Parser breaks and waits for more data
- **Unknown identifiers**: Logged as debug, data treated as text
- **Oversized buffers**: Stale data is periodically cleaned
- **Disconnection**: Reader loop catches errors, calls `updateConnectionStatus(false)`
