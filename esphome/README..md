# ESPHome AECC Battery (RS485 / Modbus)

ESPHome configuration for **AECC-platform plug-and-play batteries**, sold under brands such as Humsienk, Lunergy and Voltdeer. An ESP32-S3 reads and controls the inverter **locally over RS485 (Modbus RTU)**, with no cloud involved.

Modbus handles almost everything: live readings, all inverter settings, and the battery power setpoint. The battery's own WiFi module (JSON API, TCP port 8080) is used only for the few energy-manager settings that don't exist on Modbus.

> Register map from [makstech/esphome-aecc](https://github.com/makstech/esphome-aecc). Work modes, entity names and sign convention from [StekkerDeal/aecc-battery-local](https://github.com/StekkerDeal/aecc-battery-local).

---

## Features

- **Live readings over Modbus:** SOC, battery / AC / backup / PV / grid-port power, three temperatures, fault status, alarm flags and lifetime energy counters.
- **Local power control:** a signed **Power Setpoint** written to the inverter every second, with adjustable power and SOC limits.
- **Work Mode** dropdown: *Self-Consumption (AI)* (the battery's own logic) or *Custom / Manual* (you set the power).
- **Safe fallback:** schedule slot 3003 follows the setpoint, so a missed Modbus write doesn't change the battery power.
- **Watchdog:** if the battery stops following the setpoint, the ESP re-sends the energy-manager setup on its own.
- **All inverter settings** as numbers, switches and dropdowns: charge currents, voltages, SOC thresholds, priorities, grid code and more. Risky ones are hidden by default.
- **Web interface** on the ESP itself (port 80), with entities grouped as Info / Control / Status / Settings / Advanced / Diagnostic.
- **Status LED** on the AtomS3: green means WiFi and Home Assistant are connected, blinking green means WiFi only, red means no WiFi, blue means the fallback hotspot is active.

## Hardware

| Part | Notes |
|---|---|
| **M5Stack AtomS3 Lite** (ESP32-S3) | `esp32-s3-devkitc-1`, 8 MB flash, ESP-IDF framework |
| **RS485 transceiver** | UART on **GPIO6 (TX)** / **GPIO5 (RX)**, 9600 baud, 8N1 |
| **Cable to the inverter** | RS485 on the inverter's RJ45 port. The pins depend on your brand (see below). |

**RJ45 pinout (Modbus A / B), tested:**

| Battery | Modbus A | Modbus B |
|---|---|---|
| Voltdeer SR5000 (Pro) | pin 2 | pin 1 |
| Humsienk Nova 7.68 kWh | pin 7 | pin 8 |

<img src="images/rj45_pinout.png" alt="RJ45 pin numbering (T568B colours), tab down" width="320">

> ⚠️ **Wiring:** the RJ45 carries more than one RS485 bus (host port and BMS links). The bundled CT meter also bridges nets across pins. **Don't use a straight 8-wire patch cable.** Break out only the two conductors you need. See makstech's [PROTOCOL.md](https://github.com/makstech/esphome-aecc/blob/main/docs/PROTOCOL.md).

## Installation

1. Copy `aecc_battery_rs485.yaml` into your ESPHome config folder.
2. Make sure `secrets.yaml` contains `wifi_ssid` and `wifi_password`.
3. Edit the `substitutions:` at the top (see below). At minimum, set `aecc_ip`.
4. Give the battery's WiFi module a **fixed IP / DHCP reservation** in your router.
5. Compile and flash (**ESPHome 2026.9.0 or newer**).
6. Close the vendor app (see the warning below).

> 💡 **Updating / reflashing:** set **Work Mode** to **Self-Consumption (AI)** before you flash an update. While the ESP restarts it can't write the Power Setpoint, so in *Custom / Manual* the battery would fall back to slot 3003 for that time. In *Self-Consumption (AI)* the battery runs on its own and isn't affected. Switch back to *Custom / Manual* afterwards if you need it.

## Configuration (`substitutions`)

| Substitution | Default | Meaning |
|---|---|---|
| `name` / `friendly_name` | `aecc-battery` | Device name |
| `aecc_ip` | – | IP address of the **battery's** WiFi module (JSON API) |
| `aecc_port` | `8080` | JSON API port |
| `ems_slot_min_interval` | `10` | Minimum seconds between JSON updates of schedule slot 3003 |
| `ems_min_soc` | `15` | Starting value of **Discharge Limit** (%) |
| `ems_max_soc` | `100` | Starting value of **Charge Limit** (%) |
| `ems_max_charge` | `2400` | Hardware maximum charge power (W): upper end and default of **Max Charge Power** |
| `ems_max_discharge` | `2400` | Hardware maximum discharge power (W): upper end and default of **Max Discharge Power** |

## How power control works

The inverter only obeys the Modbus setpoint (register `0xFE16`) while its **energy manager** runs in custom mode with an active schedule slot. Those energy-manager settings exist **only** in the JSON API on port 8080, so the ESP uses both channels:

| What | Channel | When |
|---|---|---|
| Power Setpoint → `0xFE16` | **Modbus** | Every second, in *Custom / Manual* |
| All readings and inverter settings | **Modbus** | Readings every 5 s, temperatures every 30 s, settings every 10 min |
| Work Mode, energy-manager setup (3000, 3020–3030), slot 3003, SOC limits (3023/3024), power cap (3039) | **JSON** | On a mode change, 20 s after boot, every 5 min, when a limit changes, and when the watchdog fires |
| Slot 3003 follows the setpoint | **JSON** | Once the setpoint has been stable for 2 s, at most every `ems_slot_min_interval` s |

If a Modbus write is missed, the battery falls back to slot 3003 after about 3 seconds. Because that slot holds the current setpoint, nothing changes.

### Work modes

| Mode | JSON registers written |
|---|---|
| **Self-Consumption (AI)** | 3000=1, 3020=3, 3021=1, 3022=1, 3030=0, 3023/3024 = SOC limits, slot 3003 cleared |
| **Custom / Manual** | 3000=1, 3020=6, 3021=0, 3022=0, 3026=0, 3029=0, 3030=1, 3023/3024 = SOC limits, 3039 = max power, slot 3003 = setpoint |

The selected mode survives a reboot. *Custom / Manual* is sent to the battery again 20 seconds after boot.

### Sign convention

Following StekkerDeal: **positive = charging, negative = discharging.** This applies to **Power Setpoint**, **Battery Power** and **Active Power Setpoint**. The ESP converts the sign for the inverter, which itself uses + = discharge.

## Entities

### Control

| Entity | Type | Description |
|---|---|---|
| Work Mode | select | Self-Consumption (AI) / Custom / Manual |
| Power Setpoint | number | W, + = charge, − = discharge, 0 = idle (Custom / Manual only) |
| Max Charge Power / Max Discharge Power | number | W; caps the setpoint and sets the battery's power cap (3039) |
| Charge Limit / Discharge Limit | number | %, SOC window (50–100 / 5–50) |
| Read EMS Config | button | Reads the energy-manager settings over JSON |
| EMS Reply | text | Last successful JSON reply |
| EMS Errors | sensor | JSON requests that failed after 3 attempts |
| Setpoint Fallbacks | sensor | Readings where `0xFE16` didn't hold the setpoint |
| Device Power | switch | Switches the whole inverter off (hidden by default) |

### Info

Battery SOC · Battery Power · Battery Charging Power · Battery Discharging Power · AC Charging Power · Grid Power (inverter's own grid port, + = export) · Backup Power · PV Power · Active Power Setpoint · Available Charge Power · Lifetime Energy Charged · Lifetime Energy Discharged · Lifetime Energy to Grid · Lifetime RTE (round-trip efficiency, %)

### Status

Fault (65039) · PV Radiator Temperature · Inverter Temperature · Transformer Temperature

### Settings / Advanced

All inverter settings from the makstech register map (`0xA028`–`0xA0AF`, `0x9C40`, `0x9ACE`): charge currents, CV/float/cut-off voltages, SOC thresholds, output voltage and frequency, inverter mode, charge priority, on-grid mode, grid-tie power cap, and more.

These entities are **hidden by default**, because a wrong value can damage the pack or break grid compliance: **Anti Islanding, BMS Function, Parallel Mode, Battery Type, BMS Protocol, Grid Standard**.

### Diagnostic

Nominal Power · Nominal Battery Power · Alarm Flags · Atom temperature · ESP WiFi Signal Strength / Percentage · UART Logging (switch)

## Important notes

- **One JSON client at a time.** The battery's WiFi module serves only one connection. **Close the vendor app** and disable any other Home Assistant integration that talks to port 8080. Otherwise requests get no reply ("no reply" errors in EMS Reply / EMS Errors).
- **The vendor app's AI mode overwrites the energy-manager settings.** In *Custom / Manual*, the ESP re-sends them every 5 minutes.
- **Enum values can differ per firmware build.** For example, *BMS Protocol* reads **18** on the author's unit, a value not in the published list, so it's added as "Factory protocol 18". The dropdowns show **Unknown** (and log the raw value) for anything they don't recognise, and never write that value back.
- **Settings are only verified on a few units.** Read a value back before trusting it. *Mains Max Charge Current* (`0xA031`) is known not to match the app.
- **Local control may be capped at 800 W** unless 3039 is raised. The ESP sets 3039 to the higher of Max Charge Power and Max Discharge Power.

## Modbus timing

| Setting | Value | Why |
|---|---|---|
| `send_wait_time` | 350 ms | Reply window. The inverter can answer slowly, and shorter windows lose frames. |
| `turnaround_time` | 15 ms | The inverter needs at least 12 ms between frames. |
| Telemetry / temperatures / settings | 5 s / 30 s / 10 min | Keeps the bus quiet between the 1 s setpoint writes |

## Credits

- [makstech/esphome-aecc](https://github.com/makstech/esphome-aecc): Modbus register map and protocol notes
- [StekkerDeal/aecc-battery-local](https://github.com/StekkerDeal/aecc-battery-local): JSON API, work modes, entity names, sign convention and the extra Modbus telemetry registers

## Disclaimer

This project isn't affiliated with AECC or any battery brand. Writing settings to your inverter is at your own risk. Wrong values can stop the battery, damage the pack, or break grid-connection rules. Use the hidden *Advanced* settings only if you know what they do.
