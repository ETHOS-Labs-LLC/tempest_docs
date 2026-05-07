# Image Transfer

## Overview

Image transfer over radio is the most complex operation in the TEMPEST system. Images are captured by the Pi Zero camera, compressed, encoded, chunked, and transmitted packet-by-packet. The ground station reassembles, decompresses, and displays the image.

## Transfer Pipeline

```
┌─────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ rpicam-still│───►│   gzip   │───►│  base64  │───►│  chunk   │───►│  radio   │
│  (JPEG)     │    │ compress │    │  encode  │    │ + metadata│   │   TX     │
└─────────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘

                            SATELLITE (Pi Zero)
═══════════════════════════════════════════════════════════════════════════════
                            GROUND STATION (Browser)

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  radio   │───►│  parse   │───►│ reassemble│──►│  base64  │───►│  gunzip  │
│   RX     │    │ metadata │    │  chunks  │    │  decode  │    │ decompress│
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                                     │
                                                              ┌──────▼──────┐
                                                              │  Display    │
                                                              │  JPEG       │
                                                              └─────────────┘
```

## Step 1: Capture (Satellite)

```bash
TAKE_PHOTO
```

The camera subsystem executes:

```bash
rpicam-still -q 40 -o capture/{count}.jpg
```

Then gzip compresses the JPEG. The `PHOT` response returns the filename.

## Step 2: Initiate Transfer (Ground)

```bash
SEND_IMAGE 5.jpg.gz
```

The FSW loads the gzip file and processes it into chunks.

## Step 3: Chunking (Satellite)

The file is split into chunks sized for the active radio:

| Radio | Raw Chunk Size | Base64 Size | + Metadata | Total |
|-------|---------------|-------------|------------|-------|
| RFM69 | 33 bytes | 44 chars | + 8 digits | ~52 bytes |
| RFM95 | 168 bytes | 224 chars | + 8 digits | ~232 bytes |

Each packet is formatted as:

```
SEND[base64_data][total_chunks:04d][current_chunk:04d]
```

**Example** (RFM69, 100 total chunks, chunk #7):

```
SENDaGVsbG8gd29ybGQhIFRoaXMgaXMgYSB0ZXN0IGNodW5r01000007
    ├─── 44 chars base64 ───┤├─total─┤├─current─┤
```

A `.chunked` file is also written to disk for retransmission support.

## Step 4: Paced Transmission (Satellite)

The FSW transmits packets with a configurable delay to avoid overwhelming the receiver:

- **Delay**: 0.25 seconds between packets (`DEFAULT_TX_DELAY`)
- **Progress**: Logged every 50 packets
- **End marker**: `XFRC` packet with total packet count

```
SEND packet 0 → 0.25s → SEND packet 1 → 0.25s → ... → SEND packet N → XFRC
```

## Step 5: Reception (Ground Station)

The browser processes each incoming `SEND` packet:

1. Extract the last 8 characters as metadata
2. Parse `total_chunks` (first 4 digits) and `current_chunk` (last 4 digits)
3. Store the base64 data in a `packetBuffer` Map, keyed by chunk number
4. Track received chunks in a Set

```javascript
// Metadata extraction
const metaStr = textData.substring(textData.length - 8);
const totalChunks = parseInt(metaStr.substring(0, 4));
const currentChunk = parseInt(metaStr.substring(4, 8));
const b64Data = textData.substring(0, textData.length - 8);
```

![Placeholder: Image reception progress in event log](../img/placeholder-gs-image-progress.png)

## Step 6: Completion Check

When the `XFRC` packet arrives:

1. Compare total expected packets against received count
2. If all received: trigger image reconstruction
3. If missing: list missing packet numbers and suggest retransmit command

```
Image 5.jpg.gz: 95/100 received (95.0%), 5 missing
Missing packets: 12, 34, 56, 78, 99
Use: RETRANSMIT 5.jpg.gz.chunked 12 34 56 78 99
```

## Step 7: Reconstruction (Ground Station)

When all chunks are received (or manually triggered):

1. Sort chunks by number (0 to N)
2. Concatenate base64 data (remove padding `==` from all but the last chunk)
3. Base64 decode to binary (`atob()`)
4. Verify gzip header (`0x1F 0x8B`)
5. Decompress using `DecompressionStream('gzip')`
6. Create download links for both the raw `.jpg.gz` and decompressed `.jpg`

![Placeholder: Completed image with download links in browser](../img/placeholder-gs-image-complete.png)

## Retransmission

### Automatic Detection

A 30-second timeout checker runs in the background. If no new packets arrive for 30 seconds during an active transfer, it lists missing packets and suggests a retransmit command.

### Manual Retransmit

Use the retransmit input field in the Image Management section:

```
5.jpg.gz.chunked 12,34,56,78,99
```

Or send directly as a command:

```
RETRANSMIT 5.jpg.gz.chunked 12 34 56 78 99
```

### Retransmit Batching

Large retransmit requests are split into batches to fit within the command size limit (60 bytes). Batches are sent with a 2-second delay between them. Maximum 3 retransmit attempts per image.

## Active Image Management

- **Active Images**: Shows all in-progress image receptions with packet count and completion percentage
- **Clear Buffers**: Discards all image reception state

![Placeholder: Active images panel showing transfer progress](../img/placeholder-gs-active-images.png)

## Timing Considerations

| Parameter | Value | Notes |
|-----------|-------|-------|
| TX delay | 0.25s | Between consecutive packets |
| Timeout | 30s | Before suggesting retransmit |
| Retransmit batch delay | 2s | Between retransmit batches |
| Max retransmit attempts | 3 | Per image |

For a 100-packet image on RFM69: ~25 seconds transfer time (100 × 0.25s).
