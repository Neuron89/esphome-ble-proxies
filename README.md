# esphome-ble-proxies

ESPHome Bluetooth proxy firmware configurations for room-level BLE tracking in Home Assistant, with a step-by-step guide for converting existing ESPresense nodes.

Each ESP32 runs ESPHome's `bluetooth_proxy` component and forwards BLE advertisements to Home Assistant over the encrypted native API. Home Assistant's built-in `bluetooth` integration aggregates the feeds from every proxy, so trackers such as iBeacons are located by whichever room's node hears them best. No MQTT broker or custom firmware is involved.

## What is included

- One ESPHome YAML per room in `firmwares/`, identical apart from the `substitutions` block, ready to copy for additional rooms.
- Encrypted Home Assistant API, password-protected OTA updates, and a fallback access point with captive portal for recovery.
- Active BLE scanning enabled by default.
- Diagnostic entities per node: online status, uptime, Wi-Fi signal, ESPHome version, and internal temperature.
- A `secrets.example.yaml` listing every secret the configurations reference.

## ESPHome Bluetooth proxy compared with ESPresense

| | ESPresense | ESPHome Bluetooth proxy |
|---|---|---|
| Transport | MQTT broker | Home Assistant native API (encrypted) |
| Distance | RSSI-based, per node | RSSI-based, aggregated by Home Assistant |
| Discovery | MQTT discovery | mDNS auto-discovery |
| Maintenance | Custom firmware | Standard ESPHome dashboard and OTA |
| Home Assistant integration | iBeacon Tracker | iBeacon Tracker, Bluetooth, Private BLE Device |
| Tracking targets | iBeacons and other BLE advertisers | iBeacons plus any device supported by a Home Assistant Bluetooth integration (thermometers, locks, sensors) |

## Hardware

The configurations target generic ESP32 development boards (`board: esp32dev`) and were written for ESP32-D0WD-V3 modules. Nodes are USB-powered, so Wi-Fi power saving is disabled for reliability.

| Configuration | Device name | Framework |
|---|---|---|
| `ble-proxy-kitchen.yaml` | `ble-proxy-kitchen` | ESP-IDF |
| `ble-proxy-living-room.yaml` | `ble-proxy-living-room` | ESP-IDF |
| `ble-proxy-master-bathroom.yaml` | `ble-proxy-master-bathroom` | ESP-IDF |
| `ble-proxy-master-bedroom.yaml` | `ble-proxy-master-bedroom` | ESP-IDF |
| `ble-proxy-office.yaml` | `ble-proxy-office` | Arduino |
| `ble-proxy-shop.yaml` | `ble-proxy-shop` | ESP-IDF |
| `ble-proxy-theater.yaml` | `ble-proxy-theater` | ESP-IDF |

`ble-proxy-office.yaml` is a conservative variant: Arduino framework, serial logging left on at 115200 baud, and no internal temperature sensor. Use it as the starting point if a board boot-loops on the ESP-IDF configuration.

## Getting started

### Prerequisites

- Home Assistant with the ESPHome add-on (or a standalone ESPHome installation)
- An ESP32 development board and a USB data cable
- Chrome or Edge for web flashing

### Secrets

Copy the entries from `firmwares/secrets.example.yaml` into the `secrets.yaml` used by your ESPHome dashboard and fill in real values.

| Secret | Purpose | How to generate |
|---|---|---|
| `wifi_ssid` | Wi-Fi network name | – |
| `wifi_password` | Wi-Fi password | – |
| `wifi_fallback_password` | Password for the fallback access point (minimum 8 characters) | `openssl rand -base64 16` |
| `api_encryption_key` | Home Assistant native API encryption key | `openssl rand -base64 32` |
| `ota_password` | OTA update password | `openssl rand -base64 16` |

`secrets.yaml` is excluded by `.gitignore`.

### Build and flash a node

1. Connect the ESP32 to a computer over USB.
2. In Home Assistant, open **Settings → Add-ons → ESPHome → Open Web UI**.
3. Choose **New Device**, name it to match the configuration (for example `ble-proxy-office`), and select **ESP32** as the platform.
4. Click **Edit** on the new device card and replace the generated YAML with the matching file from `firmwares/`.
5. Click **Save**, then **Install → Manual download**, and download the compiled `.bin`.
6. Open <https://web.esphome.io/>, click **Connect**, select the board's serial port, choose **Install**, and upload the `.bin`. Flashing and reboot take about 30 seconds.
7. Mount the node in its room and power it.

Later updates can be installed over the air from the ESPHome dashboard.

### Verify in Home Assistant

1. Under **Settings → Devices & Services**, adopt the discovered ESPHome device.
2. Open the **Bluetooth** integration and confirm that a new scanner is listed for the node.
3. Check a tracked BLE device: once more than one proxy hears it, its `source` attribute changes as it moves between rooms, which confirms Home Assistant is combining the feeds.

### Add a room

Copy any ESP-IDF configuration to a new file and edit the `substitutions` block:

```yaml
substitutions:
  device_name: ble-proxy-garage
  friendly_name: BLE Proxy Garage
  room: garage
```

## Migrating from ESPresense

ESPresense nodes can be reflashed in place; the hardware does not change.

1. Convert one node first as a proof of concept, following the build and flash steps above.
2. Convert the remaining nodes one at a time so that presence detection keeps working during the migration. Each conversion takes roughly ten minutes once the first is done.
3. When every node is running ESPHome, delete the stale ESPresense devices from the Home Assistant device registry and update any dashboards or automations that reference ESPresense entities.
4. Optionally replace the iBeacon Tracker integration with Private BLE Device for trackers that use resolvable private addresses.

## Design notes

**Active scanning.** `bluetooth_proxy: active: true` sends scan requests to advertisers so they return scan-response data such as device names. For iBeacons this is faster and more reliable than passive listening, and the extra BLE traffic is negligible. If a node struggles in a busy RF environment, set `active: false` for that node.

**ESP-IDF over Arduino.** ESP-IDF is more memory-efficient and supports more concurrent BLE connections than the Arduino framework. Arduino is sufficient for a basic advertisement proxy and is used in the office configuration for easier first-flash diagnosis.

**Serial logging disabled.** The ESP-IDF configurations set `baud_rate: 0`, which frees the UART pins and reduces interrupt load. Re-enable it when debugging a board over USB.

**No reboot on API disconnect.** `reboot_timeout: 0s` keeps nodes running through brief Home Assistant restarts instead of rebooting.

## Project structure

```
firmwares/
  ble-proxy-<room>.yaml    One ESPHome configuration per room
  secrets.example.yaml     Template for the ESPHome secrets file
```

## License

MIT. See [LICENSE](LICENSE).
