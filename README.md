# esphome-xiaomi-ble-clock-sync

ESPHome package for periodic BLE time synchronization of Xiaomi thermometers with a built-in clock. Sets time via BLE GATT write, UTC+3.

## Compatible devices

| Device | ESPHome sensor platform |
|---|---|
| [Xiaomi Mijia LYWSD02](https://esphome.io/components/sensor/xiaomi_ble/#lywsd02) | — (display only, no sensor platform) |
| [MHO-C303](https://esphome.io/components/sensor/xiaomi_ble/#mho-c303) | [`xiaomi_mhoc303`](https://esphome.io/components/sensor/xiaomi_ble/#mho-c303) |

Both devices share the same BLE time-sync protocol (same service and characteristic UUIDs).

**CGD1 (Cleargrass/Qingping) is not compatible** — it uses a different GATT protocol and does not respond to this characteristic.

## Requirements

- ESP32 (ESP8266 has no BLE)
- ESPHome 2022.12 or later
- `esp-idf` framework (arduino framework causes flash overflow on 4MB boards with all components)

## How to add to your config

### Step 1 — Add substitution

```yaml
substitutions:
  clock_mac: "AA:BB:CC:DD:EE:FF"   # your device MAC address
```

### Step 2 — Add the package

```yaml
packages:
  clock_sync: github://sst1xx/esphome-xiaomi-ble-clock-sync/clock_sync.yaml@main
```

### Step 3 — Ensure these are present in your config

The package requires `esp32_ble_tracker` with `active: true` and a `time` component with `id: sntp_time`. If not already in your config, add them:

```yaml
esp32_ble_tracker:
  scan_parameters:
    active: true   # required for ble_client to find the device

time:
  - platform: sntp
    id: sntp_time
    timezone: Europe/Moscow
```

If `esp32_ble_tracker` is already defined — do not add it again, but make sure `active: true` is set.

## If the device is already used as a BLE sensor

Passive sensor platforms read BLE **advertisements** via `esp32_ble_tracker` — no connection needed.  
This package uses `ble_client` to **connect** to the device for a short write, then disconnects.

The connection is managed via `switch: platform: ble_client`. BLE scanning is **not interrupted** during sync — `esp32_ble_tracker.stop_scan` / `start_scan` are not called. Passive sensor readings continue unaffected.

The only requirement: **use the same MAC address** in both the sensor platform and in the `clock_mac` substitution.

Example — MHO-C303 used as both sensor and clock:

```yaml
substitutions:
  clock_mac: "17:75:BD:52:F8:E1"

packages:
  clock_sync: github://sst1xx/esphome-xiaomi-ble-clock-sync/clock_sync.yaml@main

esp32_ble_tracker:
  scan_parameters:
    active: true

sensor:
  - platform: xiaomi_mhoc303
    mac_address: "17:75:BD:52:F8:E1"
    temperature:
      name: Room Temperature
    humidity:
      name: Room Humidity

time:
  - platform: sntp
    id: sntp_time
    timezone: Europe/Moscow
```

## Substitutions reference

| Variable | Required | Description |
|---|---|---|
| `clock_mac` | yes | BLE MAC address of your device |

## Full example

See [example.yaml](example.yaml) for a complete working configuration.

## How it works

The sync interval is hardcoded to **24 hours**. On each sync the ESP32:

1. Turns on the `ble_client` switch, which initiates a BLE connection to the device
2. Waits 3 seconds for GATT services to be ready
3. Writes 5 bytes to the time characteristic: UTC Unix timestamp (4 bytes, little-endian) + timezone offset byte (`0x03` for UTC+3)
4. Turns off the switch, which disconnects

BLE scanning continues uninterrupted throughout — the switch-based connection does not call `stop_scan` / `start_scan`.

A **Sync Clock Now** button is available in the ESPHome web interface for manual on-demand synchronization.

BLE service UUID: `ebe0ccb0-7a0a-4b0c-8a1a-6ff2997da3a6`  
Time characteristic UUID: `ebe0ccb7-7a0a-4b0c-8a1a-6ff2997da3a6`
