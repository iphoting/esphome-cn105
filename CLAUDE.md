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

Every unit runs a coil-dry cycle: after a COOL/DRY run wets the evaporator coil (detected via the climate's `on_state` trigger checking `action == CLIMATE_ACTION_COOLING/DRYING`, which the component derives from the CN105 protocol's dedicated operating-status byte — a more direct signal on this unit than compressor frequency, which is unsupported here, see [#57](https://github.com/echavet/MitsubishiCN105ESPHome/issues/57)), an OFF command is intercepted and the unit is kept running for the configured duration before actually powering off. See [#658](https://github.com/echavet/MitsubishiCN105ESPHome/issues/658).

- `on_control` is the primary implementation, not just a fast path. It fires synchronously inside `ClimateCall::perform()`, before a command ever reaches the hardware, so intercepting an OFF or abandoning an active cycle there is immediate and lag-free. Reacting to it afterwards in `on_state` instead (as an earlier version of this feature did) races the CN105 component's debounced send — `controlDelegate()` calls `publish_state()` synchronously, but the actual over-the-wire write happens ~100ms+ later from `loop()`, so `on_state` can observe stale, pre-command state on the first pass. That race caused a real regression: a presence automation resuming `COOL` mid-dry-cycle would see the abandon check miss on the first `on_state` firing and the unit would appear stuck in `FAN_ONLY`. `on_control` only fires for commands sent via ESPHome/HA, though — an OFF (or any other change) from the unit's own IR remote is applied directly by the component (via `publish_state()`) and never reaches `on_control` — see [#658's IR-remote feedback](https://github.com/echavet/MitsubishiCN105ESPHome/issues/658). `on_state` is kept as a narrow fallback for exactly that gap: continuous wet/real-mode latching (there's no `ClimateCall` for a hardware-derived signal, so it can only be observed there), plus its own copy of OFF-interception and cycle-abandonment for the IR-remote case. Because `on_control` already clears the active-cycle flag synchronously for any HA/ESPHome command, `on_state`'s "active cycle" branch is only ever reached when the IR remote caused the change.
- Two dry-cycle methods, selected via the `coil_dry_method` substitution. Both force `fan_mode` to `AUTO` for the duration of the cycle; the real fan speed is saved beforehand. `cool_thermo_off` also bumps the setpoint, with the real value saved the same way.
  - `"fan_only"` (default) — force `FAN_ONLY`. Validated on single-outdoor-unit setups (all 5 rooms here).
  - `"cool_thermo_off"` — stay in `COOL` with the setpoint bumped to `coil_dry_target_c` (default `"30"`) instead, so a shared MXZ multi-zone outdoor unit isn't disrupted for sibling zones still calling for cooling. Unverified against real multi-zone hardware — see [#658 discussion](https://github.com/echavet/MitsubishiCN105ESPHome/issues/658#issuecomment-5140807883).
  - The saved mode, fan speed, and setpoint are restored both when the cycle finishes normally *and* when it's abandoned early by a new command. On the `on_control` abandon path (HA/ESPHome commands), restoration is exact: a field is restored only if the abandoning `ClimateCall` didn't explicitly set it (`get_mode()`/`get_fan_mode()`/`get_target_temperature()` all read as `nullopt` for an untouched field), so a field the call *did* set is always left alone. `on_state`'s IR-remote abandon path has no `ClimateCall` to inspect and falls back to a heuristic instead: a field is restored only if it still shows the dry-cycle-forced value, meaning the remote's own command probably didn't touch it.
  - The real HVAC mode (`COOL`/`DRY`) is also restored on normal cycle completion, sent in its own call right before the final OFF. This matters because the CN105 component's `OFF` handling never re-sends a mode byte — it only sends a power-off bit, leaving whatever mode byte was last transmitted (the dry-cycle-forced one) as what the unit's own memory resumes into when next powered on from its own remote. Without this restore step, the unit would come back up in `FAN_ONLY` (or `COOL` at `coil_dry_target_c`) instead of the room's real prior mode.
- Duration is tuneable live from HA via the `<name> Coil Dry Duration` number entity (1–120 min, template `number:`, `optimistic`/`restore_value`) — `on_control`/`on_state` read its current value at cycle-start, not a compile-time constant. The `coil_dry_duration_min` substitution (default `"30"`) only seeds that entity's `initial_value`; changing the substitution after first flash has no effect since `restore_value` preserves whatever was last set in HA.
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
