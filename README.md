# ESP32-C6 AdBlock

A DNS-based ad and tracker blocker adapted for the **ESP32-C6**, initially targeting the **Waveshare ESP32-C6-Pico**.

This project filters DNS queries on the local network. Devices that use the ESP32 as their DNS resolver receive `0.0.0.0` for blocked domains and real answers for everything else. It is a network-level filter, not a browser extension: it cannot remove ads served from the same domain as legitimate content, and it cannot affect traffic that does not use DNS.

This repository is adapted from the original ESP32-C3 project by M-Abozaid: https://github.com/M-Abozaid/esp32-c3-adblock

## Features

Verified in the current source:

- DNS query parsing and forwarding to an upstream resolver (Quad9 by default).
- Domain blocking using a sorted 40-bit FNV-1a hash table stored in flash.
- Parent-domain matching, so a blocked domain also blocks its subdomains.
- Custom blocked domains stored on LittleFS; allow rules are applied at blocklist build time.
- Per-client statistics and a banned-client list.
- Captive-portal Wi-Fi provisioning (open access point + web form).
- Web dashboard with system metrics (IP, RSSI, temperature, free heap, uptime).
- HTTP Basic Authentication and CSRF protection on state-changing endpoints.
- Blocklist upload and remote auto-update over Wi-Fi.
- Firmware OTA over Wi-Fi.
- Non-blocking DNS forwarding with up to 8 pending upstream queries.
- Wi-Fi reconnection with bounded backoff.
- Task watchdog (8 s) and reset-reason logging.
- EDNS query forwarding (upstream replies preserved).

Implemented but not yet verified on physical hardware beyond the initial port:

- OTA end-to-end after the C6 stability changes.
- Captive portal under the new watchdog configuration.
- Thermal behaviour under sustained load.

## Hardware compatibility

Tested hardware:

| Board | MCU | Flash | Status |
|---|---|---|---|
| Waveshare ESP32-C6-Pico | ESP32-C6FH4 | 4 MB | Verified |

Other ESP32-C6 boards with 4 MB flash and a native USB Serial/JTAG port may work, but only the Waveshare board above has been tested. The ESP32-C6 supports 2.4 GHz Wi-Fi only; it cannot join a 5 GHz-only network.

## Requirements

- Waveshare ESP32-C6-Pico (or compatible ESP32-C6 board with 4 MB flash)
- USB-C data cable (not charge-only)
- Git
- Python 3
- PlatformIO Core

Install PlatformIO Core using the official documentation: https://docs.platformio.org/en/latest/core/installation/index.html

On macOS, PlatformIO can be installed with pipx:

```bash
brew install pipx
pipx install platformio
pipx ensurepath
```

## Clone the repository

```bash
git clone https://github.com/neohackt/esp32-c6-adblock.git
cd esp32-c6-adblock
```

## Configure Wi-Fi safely

The tracked template is `src/secrets.example.h`. Copy it locally and edit the copy:

```bash
cp src/secrets.example.h src/secrets.h
```

Edit `src/secrets.h` and set at least `WEB_USER`, `WEB_PASS`, and `OTA_PASS` to non-placeholder values. `WIFI_SSID` and `WIFI_PASS` are optional: leave the placeholders and use the on-device captive portal instead.

`src/secrets.h` is gitignored and must never be committed. Do not commit real credentials.

If no valid Wi-Fi credentials are present, the device starts an open access point named `C6-AdBlock-XXXX` with a captive portal at `http://192.168.4.1`. Join it from a phone, select your network, and enter the password. The device then restarts and joins that network.

## Compile the firmware

Build for the ESP32-C6:

```bash
pio run -e c6
```

Other supported environments:

```bash
pio run -e c3        # original ESP32-C3 target
pio run -e esp32dev  # classic ESP32
```

A successful build ends with `SUCCESS` and reports RAM and flash usage. The C6 firmware is written to `.pio/build/c6/firmware.bin`.

## Flash the firmware

Identify the serial port:

```bash
pio device list
```

On macOS the Waveshare board appears as `/dev/cu.usbmodemXXXX` (USB JTAG/serial debug unit). On Windows it appears as a `COM` port. Use the actual port reported by your system; do not guess.

Upload firmware:

```bash
pio run -e c6 -t upload --upload-port /dev/cu.usbmodemXXXX
```

If upload fails, hold BOOT, press and release RESET, then release BOOT to enter download mode, and retry.

Open the serial monitor:

```bash
pio device monitor -p /dev/cu.usbmodemXXXX -b 115200
```

The startup output includes the assigned IP address and the dashboard URL.

## Upload LittleFS and the blocklist

Firmware and filesystem are separate artifacts. The blocklist lives in the LittleFS partition.

Build the filesystem image (includes the default blocklist if `data/blocklist.bin` exists):

```bash
pio run -e c6 -t uploadfs --upload-port /dev/cu.usbmodemXXXX
```

Generate the default blocklist first if needed:

```bash
python3 tools/build_blocklist.py data/blocklist.bin
```

This downloads the StevenBlack hosts list and Hagezi Light list and produces a sorted 40-bit hash blob. The default build contains roughly 119,000 domains and is about 0.57 MB. The upstream list licenses apply to the generated data.

Confirm the blocklist loaded by checking the serial output for `blocklist: N domains (valid)` or the dashboard's domain count. If `/blocklist.bin` is missing or invalid, filtering is disabled and all queries are forwarded.

You can also upload a new blocklist later through the dashboard's Blocklist upload form without reflashing.

## First startup

On boot the device prints `[c6-adblock] booting`, the reset reason, and blocklist status. If Wi-Fi is unconfigured it starts the captive portal. Otherwise it connects to the configured network and prints its IP address and `dashboard: http://c6adblock.local`.

mDNS makes the device reachable as `c6adblock.local` on most networks.

## Verify DNS blocking

Query the device directly:

```bash
dig @<device-ip> doubleclick.net A +noedns
dig @<device-ip> googlesyndication.com A +noedns
dig @<device-ip> freeconvert.com A +noedns
```

Expected: blocked domains resolve to `0.0.0.0`; ordinary domains resolve to real IP addresses. Exact blocklist contents depend on the generated list.

Test EDNS:

```bash
dig @<device-ip> example.com A +bufsize=1232
```

These tests query the ESP32 directly. Normal devices on your network will not use it until you configure their DNS settings.

## Configure devices to use the ESP32

Set a device's DNS server to the ESP32's IP address, or set your router's DHCP DNS option to that IP. The ESP32 must be powered and connected to Wi-Fi for DNS to work; if it is unavailable, clients that depend on it will lose DNS.

Recommended: assign a DHCP reservation to the ESP32 so its IP does not change.

Note that DNS-over-HTTPS and other encrypted DNS methods in browsers or operating systems can bypass local DNS filtering.

On macOS, set DNS in System Settings -> Network -> (interface) -> Details -> DNS.

On Windows, set DNS in Settings -> Network & Internet -> (adapter) -> DNS server assignment.

On Android, set DNS in Wi-Fi network details -> IP settings -> Static.

## Dashboard

Address: `http://c6adblock.local` or `http://<device-ip>`.

Visible statistics: total blocked, total allowed, blocklist size, client count, Wi-Fi RSSI, temperature, free RAM, uptime, and per-client blocked/allowed counts.

State-changing endpoints (ban, add block, upload, OTA, settings) require HTTP Basic Authentication using `WEB_USER` and `WEB_PASS` from `src/secrets.h`, plus an `X-Requested-With: c3-adblock` header. Read-only views (`/`, `/stats.json`) are open.

## Troubleshooting

- Device not detected over USB: use a data-capable cable; try BOOT+RESET to enter download mode.
- Upload failures: confirm the port, reduce upload speed, or enter download mode manually.
- Wi-Fi connection failures: check credentials, 2.4 GHz availability, and RSSI; use the captive portal.
- Missing or invalid blocklist: serial shows `blocklist: MISSING` or `INVALID`; run `uploadfs` or upload via dashboard.
- DNS timeouts: check Wi-Fi stability, upstream reachability, and that the device is not overloaded.
- Domains not blocked: verify the domain is in the generated list; parent matching covers subdomains only.
- DNS requests bypassing the ESP32: check client/router DNS settings and DNS-over-HTTPS.
- EDNS issues: the device handles EDNS forwarding; blocked replies drop the OPT record, which is standards-compliant.

## Security and privacy

- `src/secrets.h` is gitignored; never commit real credentials.
- Placeholder `WEB_PASS` or `OTA_PASS` values trigger a serial warning and dashboard banner.
- Dashboard traffic is plain HTTP on port 80; Basic Auth credentials are base64-encoded and visible to anyone who can sniff LAN traffic.
- OTA and dashboard mutating endpoints require authentication and a CSRF header.
- The access point used for provisioning is open by design during the setup window.

## Limitations

DNS filtering cannot reliably remove every advertisement. Ads served from first-party domains, certain video platforms, and content delivered over HTTPS from the same domain as legitimate content may not be blocked. This project does not perform HTTPS inspection or content filtering.

## Development and testing

Build:

```bash
pio run -e c6
```

Unit tests:

```bash
python3 -m unittest discover -s tools -p 'test_*.py' -v
```

Report bugs at https://github.com/neohackt/esp32-c6-adblock/issues.

## License and attribution

This project is licensed under the MIT License. See [LICENSE](LICENSE).

It is adapted from the ESP32-C3 project by M-Abozaid: https://github.com/M-Abozaid/esp32-c3-adblock. Original copyright and attribution are preserved in the LICENSE file.

## Tested environment

| Component | Version |
|---|---|
| macOS | 27.0.1 (arm64) |
| PlatformIO Core | 6.2.0 |
| espressif32 platform | 8.0.0 |
| Arduino framework | 3.x (ESP32-C6) |
| Python | 3.15 |
| Waveshare ESP32-C6-Pico | ESP32-C6FH4, 4 MB flash |

Unverified functionality is marked as such in the Features section.
