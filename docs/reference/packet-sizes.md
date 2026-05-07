# Packet Size Reference

## Radio Payload Limits

| Radio | FIFO Size | Adafruit Header | `\n` Delimiter | Usable Payload |
|-------|-----------|-----------------|----------------|----------------|
| RFM69 | 66 bytes | 4 bytes | 1 byte | **61 bytes** |
| RFM95 | 252 bytes | 4 bytes | 1 byte | **247 bytes** |

The configured chunk sizes in `config.yml` provide additional margin:

| Radio | Config `chunk_size` | Effective Payload |
|-------|--------------------|--------------------|
| RFM69 | 50 bytes | 50 - 1 (`\n`) = **49 bytes data** |
| RFM95 | 240 bytes | 240 - 1 (`\n`) = **239 bytes data** |

## Telemetry Packet Sizes (including 4-byte ID)

All sizes include the 4-byte identifier but exclude the `\n` appended by `radio.py`.

| Packet | Total Size | Fits RFM69? | Fits RFM95? |
|--------|-----------|-------------|-------------|
| GYRO | 16 | Yes | Yes |
| ACCL | 16 | Yes | Yes |
| MAGN | 16 | Yes | Yes |
| GRAV | 16 | Yes | Yes |
| EULR | 16 | Yes | Yes |
| BME2 | 16 | Yes | Yes |
| TEMP | 8 | Yes | Yes |
| QUAT | 20 | Yes | Yes |
| ADCS | 32 | Yes | Yes |
| SOLR | 36 | Yes | Yes |
| EPSS | 28 | Yes | Yes |
| OBCC | 8 | Yes | Yes |
| OBCR | 8 | Yes | Yes |
| OBCD | 8 | Yes | Yes |
| HOST | 15 | Yes | Yes |
| BECN | 24 | Yes | Yes |
| XFRC | 8 | Yes | Yes |

## Chunked Data Sizes

For multi-packet responses (`OBCP`, `OBCL`), each chunk is:

```
[4-byte ID] + [payload] + [\n from radio.py]
max_chunk_payload = chunk_size - 5
```

| Radio | chunk_size | ID | `\n` | Max Payload per Chunk |
|-------|------------|----|----|----------------------|
| RFM69 | 50 | 4 | 1 | **45 bytes** |
| RFM95 | 240 | 4 | 1 | **235 bytes** |

## Image Packet Sizes

Image packets use a different calculation because they're text-based:

```
SEND[base64_data][total:04d][current:04d]
  4  +  b64_len  +    4    +     4      = total
```

| Radio | Raw Chunk | Base64 | + ID + Meta | Total | Within Limit? |
|-------|-----------|--------|-------------|-------|---------------|
| RFM69 | 33 bytes | 44 chars | + 4 + 8 = 56 | **56 bytes** | Yes (< 61) |
| RFM95 | 168 bytes | 224 chars | + 4 + 8 = 236 | **236 bytes** | Yes (< 247) |

## Size Calculation Examples

### RFM69 Image Packet

```
Raw data chunk:   33 bytes
Base64 encoded:   ceil(33/3) * 4 = 44 characters
SEND identifier:  4 bytes
Metadata:         8 bytes (4 total + 4 current)
                  ─────────
Packet total:     56 bytes
+ \n from radio:  1 byte
                  ─────────
On-air total:     57 bytes  (< 66 byte FIFO limit)
```

### RFM69 OBCP Chunk

```
OBCP identifier:  4 bytes
Text payload:     45 bytes max
                  ─────────
Packet total:     49 bytes
+ \n from radio:  1 byte
                  ─────────
On-air total:     50 bytes  (= chunk_size config)
```

## Exceeding Limits

If a packet exceeds the radio FIFO size, the Adafruit library raises an `AssertionError`. This most commonly occurs with:

- Image packets when chunk sizes aren't properly tuned
- OBC process/file listing chunks when using the wrong default size
- Any packet that doesn't account for the 4-byte Adafruit header + 1-byte `\n`

The `chunk_data()` function in `main.py` automatically calculates the correct max payload size based on the active radio's chunk size configuration.
