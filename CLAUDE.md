# MitsubishiCN105ESPHome

ESPHome configurations for controlling Mitsubishi split AC units via the CN105 serial connector, running on M5Stack NanoC6 (ESP32-C6) devices, integrated with Home Assistant.

## Hardware

- **MCU:** M5Stack NanoC6 (ESP32-C6, 4 MB flash, esp-idf framework)
- **AC protocol:** Mitsubishi CN105 serial (UART, 2400 baud, 3.3 V logic on GPIO1/GPIO2)
- **External component:** [echavet/MitsubishiCN105ESPHome](https://github.com/echavet/MitsubishiCN105ESPHome)

### Wiring (Grove port → CN105)

| Grove pin | CN105 pin | Signal |
|-----------|-----------|--------|
| 1 (Yellow / IO2) | 4 | TX from unit → RX for ESP |
| 2 (White / IO1)  | 5 | RX into unit ← TX from ESP |
| 3 (Red / 5V)     | 1 | 5 V rail (via unit's regulator) |
| 4 (Black / GND)  | 2 | GND |
| —                | 3 | 3.3 V — **not connected** |

GPIO2 RX pin requires `pullup: true` for reliable operation on ESPHome 2025.x.

## Room-to-file mapping

| File | Device name | Location | Remote temp sensor |
|------|-------------|----------|--------------------|
| `ac-br2.yaml` | ac-br2 | Bedroom 2  | `sensor.br2_temperature_mean` |
| `ac-br3.yaml` | ac-br3 | Bedroom 3  | `sensor.br3_temperature_mean` |
| `ac-dr.yaml`  | ac-dr  | Dining Room | `sensor.lr_temperature_sensor_mean` (shared with LR — no dedicated DR sensor) |
| `ac-lr.yaml`  | ac-lr  | Living Room | `sensor.lr_temperature_sensor_mean` |
| `ac-mb.yaml`  | ac-mb  | Master Bedroom | `sensor.mb_temperature_sensor_mean` |

## Secrets setup

```bash
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your real values
# Generate a fresh encryption_key:
esphome generate-key
```

`secrets.yaml` is excluded from git. Never commit it. See `secrets.yaml.example` for required keys.

## Common ESPHome commands

```bash
# Compile only
esphome compile ac-lr.yaml

# Compile and flash over USB (first time)
esphome upload ac-lr.yaml

# Flash over WiFi (OTA)
esphome upload --device <ip-or-hostname> ac-lr.yaml

# Stream logs
esphome logs ac-lr.yaml

# Flash all units (OTA)
for f in ac-*.yaml; do esphome upload "$f"; done
```

## Adding a new AC unit

1. Copy an existing config: `cp ac-lr.yaml ac-newroom.yaml`
2. Change the three substitution variables at the top:
   ```yaml
   substitutions:
     name: ac-newroom
     friendly_name: New Room AC
     remote_temp_sensor: sensor.newroom_temperature_mean
   ```
3. Add `secrets.yaml` entry if a new `encryption_key` is needed (each device should have its own).

## Networking

All units default to WiFi. To switch a unit to OpenThread/Thread:
1. Comment out the `wifi:` and `captive_portal:` blocks.
2. Uncomment the `network:` and `openthread:` blocks.
3. Add `thread_tlv` to `secrets.yaml`.

## Samples directory

`Samples/` contains upstream reference configurations for alternative hardware variants (Atom S3 Lite, NanoC6 heat pump). Do not flash these directly — they are reference only.
