# Troubleshooting

## Radio Issues

### AssertionError on Send

**Symptom:** `AssertionError` from Adafruit library when transmitting.

**Cause:** Packet exceeds radio FIFO size (66 bytes for RFM69, 252 bytes for RFM95).

**Fix:** Check packet size calculation. The total on-air size is:

```
[4-byte Adafruit header] + [your payload] + [1-byte \n from radio.py]
```

For RFM69, your payload must be <= 61 bytes.

### No Response from Satellite

**Symptom:** Commands sent but no telemetry received.

**Possible Causes:**

1. **Radio type mismatch** — GS and satellite must use the same radio type (both RFM69 or both RFM95)
2. **Frequency mismatch** — Uplink and downlink frequencies must match between GS and satellite
3. **Satellite not running** — Check `systemctl status tempest_fsw`
4. **Radio hardware issue** — Check SPI wiring and GPIO connections

### Missed Packets During Image Transfer

**Symptom:** Image transfer completes with missing packets.

**Cause:** RFM69 has higher packet loss at faster TX rates. USB CDC can also drop data under high throughput.

**Fix:**

- Increase `DEFAULT_TX_DELAY` in `main.py` (default 0.25s)
- Use retransmit: `RETRANSMIT filename.chunked pkt1 pkt2 ...`
- For persistent issues, consider switching to RFM95 (LoRa) for image transfers

## Browser Issues

### WebSerial Not Available

**Symptom:** "Connect" button doesn't show port selection dialog.

**Cause:** WebSerial API not supported or not in secure context.

**Fix:**

- Use Chrome 89+ or Edge 89+
- Must be served over HTTPS or localhost
- Firefox and Safari do not support WebSerial

### Telemetry Not Updating

**Symptom:** Event log shows packets arriving but telemetry values stay at `--`.

**Cause:** Element ID mismatch between HTML and JavaScript.

**Fix:** Verify that the HTML element IDs match what the telemetry unpackers expect:

| Element ID | Updated By |
|------------|------------|
| `cpuUsage` | OBCC, BECN |
| `ramUsage` | OBCR, BECN |
| `diskUsage` | OBCD, BECN |
| `temperature` | BME2, TEMP, BECN |
| `batteryVoltage` | EPSS |
| `epsStatus` | EPSS |
| `epsCh1`-`epsCh4` | EPSS |
| `solarXn`, `solarXp`, `solarYn`, `solarYp` | SOLR |
| `solarTotal` | SOLR |

### Fragmented Packets in Browser

**Symptom:** Garbled data or partial packet parsing.

**Cause:** USB CDC does not preserve packet boundaries. A single radio packet may arrive as multiple USB reads.

**Fix:** The parser uses newline delimiters (`\n`) for variable-length packets and waits for complete fixed-size packets. If issues persist, check that `radio.py` appends `b'\n'` to every transmission.

### Image Not Displaying After Transfer

**Symptom:** All packets received but image doesn't render.

**Possible Causes:**

1. **Corrupted base64** — Check for non-base64 characters in the data
2. **Missing gzip header** — Verify the file is actually gzip compressed (header: `0x1F 0x8B`)
3. **Chunk ordering** — Base64 padding (`==`) should only be on the final chunk

## WiFi / AP Issues

### Can't See the Satellite's SSID

**Symptom:** After `install.sh` and reboot, your laptop doesn't see
`TEMPEST-N` in the wifi list.

**Possible Causes:**

1. **Hostname not set before install** — the AP profile bakes the hostname
   into the SSID at install time. Verify with `nmcli con show tempest-ap |
   grep 802-11-wireless.ssid`. If it's wrong, fix the hostname
   (`sudo raspi-config nonint do_hostname TEMPEST-3`) and recreate the
   profile, or edit `802-11-wireless.ssid` on the existing profile.
2. **Another wifi profile beat the AP to activation** — confirm the AP is
   active with `nmcli con show --active`. If a client profile is active
   instead, set its `connection.autoconnect` to `no` and reboot.
3. **The Pi Zero W is out of range** — the AP is 2.4 GHz only and the Pi
   antenna is weak.

### Forgot the AP Password

**Symptom:** Can't connect to the satellite's AP because the PSK was rotated.

**Fix:** Pull the SD card, mount it on a laptop, and edit
`/etc/NetworkManager/system-connections/tempest-ap.nmconnection`. Update the
`psk` field under `[wifi-security]`. Reinsert and reboot.

### `WIFI_INFO` Returns Empty SSID

**Symptom:** The `WIFI` telemetry packet arrives but the SSID field is the
hostname rather than the actual AP SSID.

**Cause:** `nmcli` query failed, falling back to `socket.gethostname()`. This
usually means NetworkManager isn't running or the `tempest-ap` profile isn't
active.

**Fix:** SSH in, check `systemctl status NetworkManager` and
`nmcli con show --active`. Restart NetworkManager if needed.

## Sensor Issues

### Stale Data on First Read

**Symptom:** First sensor command returns incorrect values, subsequent reads are normal.

**Cause:** Stale data in the RP2040 UART buffer from before the FSW started.

**Fix:** The FSW performs a throwaway `POLL` on startup to flush the buffer. If the issue persists, send any command twice — the second response will be current.

### Environmental Command Returns None

**Symptom:** Sensor command returns no data or times out.

**Possible Causes:**

1. **RP2040 not connected** — Check UART wiring (GP0/GP1 on Pico, TX/RX on Pi Zero)
2. **Wrong baud rate** — Must be 9600 on both ends
3. **I2C sensor not detected** — Check BNO055/BMP280 wiring and I2C addresses
4. **RP2040 crashed** — Send `RESET_ENV` to soft-reset the MCU

### Solar Readings All Zero

**Symptom:** `GET_SOLAR` returns 0V 0mA for all panels.

**Cause:** INA219 sensors not detected on I2C bus.

**Fix:** Verify I2C addresses (0x40, 0x41, 0x44, 0x45) and wiring. Run `i2cdetect -y 1` on the Pi Zero to scan the bus.

## EPS Issues

### EPS Command Timeout

**Symptom:** EPS commands hang or return no data.

**Cause:** Bit-banged UART communication failure with ATmega328PB.

**Fix:**

1. Verify `pigpiod` is running: `systemctl status pigpiod`
2. Check GPIO wiring: TX=GPIO27, RX=GPIO22
3. Verify ATmega328PB is powered and running

## Service Issues

### FSW Won't Start

**Symptom:** `systemctl status tempest_fsw` shows failure.

**Common Causes:**

1. **pigpiod not running** — FSW depends on it for EPS communication
2. **Missing Python packages** — Run `install.sh` again
3. **Corrupted config.yml** — Delete it and let startup restore from `.bak`
4. **Hardware not connected** — Radio SPI initialization may fail

### Config Changes Not Taking Effect

**Symptom:** Editing `config.yml` doesn't change behavior.

**Fix:** The watchdog monitors for file modifications. Verify:

1. You edited `config.yml`, not `config.yml.bak`
2. The watchdog thread is running (check FSW logs)
3. Note: On reboot, `.bak` overwrites `.yml`, so persistent changes need to go in both files
