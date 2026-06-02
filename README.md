# esphome-xiaomi-ble-clock-sync

ESPHome package for periodic BLE time synchronization of Xiaomi thermometers with a built-in clock. Sets time via BLE GATT write, UTC+3 (Europe/Moscow).

## Compatible devices

| Device | ESPHome sensor platform |
|---|---|
| [Xiaomi Mijia LYWSD02](https://esphome.io/components/sensor/xiaomi_ble/#lywsd02) | — (display only, no sensor platform) |
| [MHO-C303](https://esphome.io/components/sensor/xiaomi_ble/#mho-c303) | [`xiaomi_mhoc303`](https://esphome.io/components/sensor/xiaomi_ble/#mho-c303) |

Both devices share the same BLE time-sync protocol (same service and characteristic UUIDs).

## Requirements

- ESP32 (ESP8266 has no BLE)
- ESPHome 2022.12 or later

## How to add to your config

### Step 1 — Add substitutions

```yaml
substitutions:
  clock_mac: "AA:BB:CC:DD:EE:FF"   # your device MAC address
  clock_sync_interval: "24h"        # "5min" for testing
```

### Step 2 — Add the package

```yaml
packages:
  clock_sync: github://sst1xx/esphome-xiaomi-ble-clock-sync/clock_sync.yaml@main
```

### Step 3 — Ensure these are present in your config

The package requires `esp32_ble_tracker` and a `time` component with `id: sntp_time`. If they are not already in your config, add them:

```yaml
esp32_ble_tracker:

time:
  - platform: sntp
    id: sntp_time
    timezone: Europe/Moscow
```

If `esp32_ble_tracker` is already defined in your config — do not add it again. The package does not include it to avoid conflicts.

## If the device is already used as a BLE sensor

If your config already reads data from this device via `xiaomi_mhoc303` or another passive BLE sensor platform, you can still add time sync — they use different mechanisms:

- Passive sensor platforms (`xiaomi_mhoc303`, `xiaomi_hhccjcy01`, etc.) read BLE **advertisements** via `esp32_ble_tracker` — no connection needed.
- This package uses `ble_client` to **connect** to the device for a short write, then disconnects.

They do not conflict. The package temporarily stops the BLE scan during the write (`esp32_ble_tracker.stop_scan`) and resumes it immediately after (`esp32_ble_tracker.start_scan`). The sensor data gap during sync is equal to the connection time (typically 3–5 seconds).

The only requirement: **use the same MAC address** in both the sensor platform and in the `clock_mac` substitution.

Example — MHO-C303 used as both a sensor and a clock:

```yaml
substitutions:
  clock_mac: "17:75:BD:52:F8:E1"         # same MAC as the sensor below
  clock_sync_interval: "24h"

packages:
  clock_sync: github://sst1xx/esphome-xiaomi-ble-clock-sync/clock_sync.yaml@main

esp32_ble_tracker:                        # already present — do not duplicate

sensor:
  - platform: xiaomi_mhoc303             # reads passive BLE advertisements
    mac_address: "17:75:BD:52:F8:E1"
    temperature:
      name: Room Temperature
    humidity:
      name: Room Humidity

time:
  - platform: sntp
    id: sntp_time                         # already present — do not duplicate
    timezone: Europe/Moscow
```

## Substitutions reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `clock_mac` | yes | — | BLE MAC address of your device |
| `clock_sync_interval` | no | `24h` | How often to sync time |

## Full example

See [example.yaml](example.yaml) for a complete working configuration.

## How it works

On every interval the ESP32:
1. Stops BLE scan
2. Connects to the device
3. Writes 5 bytes to the time characteristic: UTC Unix timestamp (4 bytes, little-endian) + timezone offset byte (`0x03` for UTC+3)
4. Disconnects
5. Resumes BLE scan

BLE service UUID: `ebe0ccb0-7a0a-4b0c-8a1a-6ff2997da3a6`  
Time characteristic UUID: `ebe0ccb7-7a0a-4b0c-8a1a-6ff2997da3a6`
