# Quick Start: TEMPEST-0 on RFM69 @ 915 MHz

This guide takes one CubeSat from a freshly imaged card to a working radio link,
configured as:

| Setting | Value |
|---------|-------|
| Hostname | `TEMPEST-0` |
| Radio node address | `0` |
| Radio type | RFM69 (FSK) |
| Frequency | 915 MHz |
| Uplink / downlink | Same radio |
| Encryption | Off |

!!! warning "Use the numeral zero, not the letter O"
    The hostname must be `TEMPEST-0` — **zero**, not `TEMPEST-O`.

    The FSW derives the radio node address by parsing the text after the dash
    as an integer (`utils/radio.py`):

    ```python
    uplink.node = int(socket.gethostname().split("-")[1]) * 10
    ```

    A letter `O` cannot be parsed and the FSW crashes on startup with:

    ```
    ValueError: invalid literal for int() with base 10: 'O'
    ```

    Zero is the only hostname suffix that produces node `0`, since the parsed
    integer is multiplied by 10 (`TEMPEST-1` → node 10, `TEMPEST-2` → node 20).

---

## 1. Set the Hostname

### Option A — during imaging (recommended)

Follow [Connecting to Your TEMPEST CubeSat](connecting_to_wifi.md#imaging-a-fresh-microsd-card)
and, at the **`EDIT SETTINGS`** step, set:

- **Hostname**: `TEMPEST-0`

The AP SSID is derived from the hostname, so the unit will broadcast
`TEMPEST-0` on first boot.

### Option B — on a running unit

SSH in (see [Connecting from Your Laptop](connecting_to_wifi.md#connecting-from-your-laptop)),
then:

```bash
sudo hostnamectl set-hostname TEMPEST-0
sudo reboot
```

The `tempest-ap-ssid-sync` service installed by `install.sh` rewrites the WiFi
AP SSID from the current hostname on every boot, so the SSID follows the
rename automatically after the reboot.

### Verify

```bash
hostname
```

Expected output — confirm visually that this is a zero:

```
TEMPEST-0
```

---

## 2. Set the Radios

The radio type and frequency live in `config.yml` in the FSW directory.

!!! danger "Edit both config.yml and config.yml.bak"
    On every start, `main.py` copies `config.yml.bak` over `config.yml`:

    ```python
    if os.path.exists(BACKUP_FILE):
        shutil.copy(BACKUP_FILE, CONFIG_FILE)
    ```

    This resets any live changes made during a session (the FSW watches
    `config.yml` for runtime edits). If you only edit `config.yml`, your
    settings are **silently reverted the next time the FSW starts**. Edit the
    `.bak` file too, or copy one over the other.

Edit the file:

```bash
cd ~/TEMPEST_FSW
nano config.yml
```

Set uplink and downlink to RFM69 at 915 MHz:

```yaml
Tempest:
  Comms:
    uplink:
      radio: rfm69
      freq: 915
      encryption: false
      enc_key: ""

    downlink:
      radio: rfm69
      freq: 915
      chunk_sizes:
        rfm95: 240
        rfm69: 50
      encryption: false
      enc_key: ""

    crosslink:
      radio: "none"
      frequency: 921
      encryption: false
      enc_key: ""
```

Then push the same content into the backup so it survives a restart:

```bash
cp config.yml config.yml.bak
```

Notes on these values:

- `radio` must be lowercase `rfm69`. Any other value except `rfm95` raises
  `ValueError: Unsupported uplink radio type`.
- Because uplink and downlink are both `rfm69`, the FSW shares a single radio
  instance between them — you'll see `Downlink  same as uplink` at startup.
- Leave `chunk_sizes` alone. The RFM69 FIFO is 66 bytes, and the 50-byte chunk
  size accounts for the 4-byte library header.
- 915 MHz is the US ISM band (902–928 MHz). Confirm this band is legal in your
  region before transmitting.

---

## 3. Start the FSW

Restart the service so it picks up the new hostname and config:

```bash
sudo systemctl restart tempest_fsw
sudo systemctl status tempest_fsw
```

To watch it run in the foreground instead:

```bash
sudo systemctl stop tempest_fsw
cd ~/TEMPEST_FSW
python3 main.py
```

### Verify

The startup banner reports the live radio settings. Confirm all three lines:

```
  Radio     RFM69 @ 915 MHz
  Node      0
  Downlink  same as uplink
  Status    Ready
```

If `Node` reads anything other than `0`, the hostname is wrong — go back to
step 1. If the FSW exits with a `ValueError` about `int()`, the hostname
contains the letter `O`.

---

## 4. Match the Ground Station

The ground station will not report an error if it disagrees with the
satellite — the link just fails silently. Open the GS browser interface and
set both sides in the radio configuration panel:

**Uplink (ground → satellite)**

| Field | Value |
|-------|-------|
| Radio Type | RFM69 |
| Band | 915 MHz |
| Frequency | 915.0 |
| Satellite Node | `0` |

**Downlink (satellite → ground)**

| Field | Value |
|-------|-------|
| Radio Type | RFM69 |
| Band | 915 MHz |
| Frequency | 915.0 |
| Ground Node | `255` |

Click **Apply Configuration**. The GS sends:

```
set_uplink_radio rfm69 915.0 0
set_downlink_radio rfm69 915.0 255
```

This addresses ground transmissions to node `0` (your satellite) while the GS
itself listens as node `255`. The FSW replies to whichever node sent the
command, so responses come back to `255` automatically.

See [Radio Configuration](ground-station/radio-config.md) for the full panel
reference.

---

## 5. Confirm the Link

From the ground station, send a command that needs no sensors:

```
GET_HOSTNAME
```

On the satellite console you should see the exchange:

```
  14:22:07  RX node 255  GET_HOSTNAME  → TX 15 bytes → node 255  OK
```

The ground station receives a `HOST` packet whose payload is `TEMPEST-0`
padded to 11 bytes. If that round-trips, the link is up.

Try one more with live data:

```
OBC_CPU
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| FSW exits with `ValueError: invalid literal for int()` | Hostname uses letter `O`, or has no `-N` suffix | `sudo hostnamectl set-hostname TEMPEST-0` and reboot |
| `Node` shows a value other than `0` | Hostname suffix isn't `0` | Rename the host; remember the suffix is multiplied by 10 |
| `ValueError: Unsupported uplink radio type` | `radio:` isn't `rfm69` or `rfm95` | Check spelling and case in `config.yml` |
| Settings revert after every restart | `config.yml.bak` still holds the old values | `cp config.yml config.yml.bak` |
| Startup shows RFM69 @ 915 but no packets arrive | GS is on a different radio type, frequency, or node | Re-apply step 4; mismatches fail silently |
| Commands arrive but no response reaches the GS | GS node doesn't match the address the FSW replies to | Set the GS **Ground Node** to `255` |
| Nothing at all on either end | RFM69 wiring / encryption mismatch | Confirm `encryption: false` on both ends |

For more, see [Troubleshooting](reference/troubleshooting.md).

---

## Fleet Note

Node addresses are `hostname_suffix × 10`, so a fleet reads:

| Hostname | Node |
|----------|------|
| `TEMPEST-0` | 0 |
| `TEMPEST-1` | 10 |
| `TEMPEST-2` | 20 |
| `TEMPEST-25` | 250 |

Every unit in radio range needs a unique suffix — duplicates collide on the
air. Note that `TEMPEST-0` and a GS ground node of `0` would also collide, so
keep the ground station at `255`.
