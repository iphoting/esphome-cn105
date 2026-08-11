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

## Coil-dry ("gym sock" smell prevention)

Every unit runs a coil-dry cycle: after a COOL/DRY run wets the evaporator coil (detected via the climate's `on_state` trigger checking `action == CLIMATE_ACTION_COOLING/DRYING`, which the component derives from the CN105 protocol's dedicated operating-status byte — a more direct signal on this unit than compressor frequency, which is unsupported here, see [#57](https://github.com/echavet/MitsubishiCN105ESPHome/issues/57)), an OFF command is intercepted and the unit is kept running for `coil_dry_duration_min` minutes before actually powering off. See [#658](https://github.com/echavet/MitsubishiCN105ESPHome/issues/658).

- Detection and interception both live in `on_state`, not just `on_control`. `on_control` only fires for commands sent via ESPHome/HA; an OFF pressed on the unit's own IR remote is applied directly by the component (via `publish_state()`) and never reaches `on_control` — see [#658's IR-remote feedback](https://github.com/echavet/MitsubishiCN105ESPHome/issues/658). `on_control` is kept only as a fast path that rewrites an HA/ESPHome-issued OFF in flight (so the entity's state history doesn't show a phantom "Off"); `on_state` is the complete implementation — it also catches remote-issued offs (by commanding the unit back into the dry-cycle mode after the fact) and cancels an active cycle if the reported state stops matching what the cycle forced (whether that change came from HA or the remote).
- Two dry-cycle methods, selected via the `coil_dry_method` substitution. Both force `fan_mode` to `AUTO` for the duration of the cycle; the real fan speed is saved beforehand. `cool_thermo_off` also bumps the setpoint, with the real value saved the same way.
  - `"fan_only"` (default) — force `FAN_ONLY`. Validated on single-outdoor-unit setups (all 5 rooms here).
  - `"cool_thermo_off"` — stay in `COOL` with the setpoint bumped to `coil_dry_target_c` (default `"30"`) instead, so a shared MXZ multi-zone outdoor unit isn't disrupted for sibling zones still calling for cooling. Unverified against real multi-zone hardware — see [#658 discussion](https://github.com/echavet/MitsubishiCN105ESPHome/issues/658#issuecomment-5140807883).
  - The saved fan speed and setpoint are restored both when the cycle finishes normally *and* when it's abandoned early by a new command — but on the abandon path, a field is only restored if it still shows the dry-cycle-forced value, meaning the abandoning command didn't touch it. A field the abandoning command explicitly set (e.g. the remote picking a new fan speed) is left alone rather than clobbered, since `on_state` can only see the resulting state, not which fields the command actually set.
- Tune per-room duration via the `coil_dry_duration_min` substitution (default `"30"`).
- Toggle the feature off per-room with the `<name> Coil Dry Enabled` switch in HA.
- `<name> Coil Dry Active` / `<name> Coil Dry Remaining` diagnostic entities show cycle status.
- Sending any new command while a dry cycle is active (e.g. turning cooling back on, via HA or the remote) immediately cancels the cycle.

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
