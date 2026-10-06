# Voltdeer SR5000 Pro / AECC battery → HBC (local TCP)

Home Assistant package for the **Voltdeer SR5000 Pro** and other batteries on the **AECC platform**, driven over the battery's **local TCP API** via the [AECC Battery (Local TCP)](https://github.com/StekkerDeal/aecc-battery-local) integration.

Exposes the exact `marstek_m1_*` entities expected by [Home Battery Control](https://github.com/gitcodebob/marstek-venus-rs485-node-red) (HBC), so the Voltdeer can be used as the **M1** battery in an HBC setup.

No cloud dependency: all reads and writes go to the battery on your LAN.

Package file: [`aecc_battery_to_m1.yaml`](aecc_battery_to_m1.yaml) — version **1.1** (see the version history at the top of the file).

## Supported devices

| Product | Notes |
| --- | --- |
| Voltdeer SR5000 Pro | Tested. 5.12 kWh, 51.2 V nominal (defaults in the package). |
| Other AECC-platform batteries | Should work if the AECC integration supports them; set capacity / voltage helpers accordingly. |

## What is in the package

One YAML file with five sections:

| Section | Purpose |
| --- | --- |
| `template` | Fake Marstek M1 sensors, numbers and selects that HBC reads and writes. |
| `input_select` / `input_number` | Helpers storing the HBC decisions (control mode, forcible mode, charge / discharge power, charge-to-SOC) plus battery capacity and nominal voltage. |
| `utility_meter` | Daily charged / discharged energy meters based on the AECC lifetime counters. |
| `automation` | Translates the HBC decisions into one signed **Power Setpoint** on the battery. |

## Install

1. Install the HBC package, dashboard and Node-RED strategy flows as documented upstream, and configure HBC with **one battery and a P1 meter**.
2. Install the **AECC Battery (Local TCP)** integration via HACS ([StekkerDeal/aecc-battery-local](https://github.com/StekkerDeal/aecc-battery-local)). Select brand **Voltdeer** during setup.
3. Keep the device friendly name **AECC Battery** (integration default) so the entities are named `sensor.aecc_battery_*`, `number.aecc_battery_power_setpoint`, `select.aecc_battery_work_mode`. If you used another name, search/replace the `aecc_battery` prefix in the YAML (see [Configuration](#configuration)).
4. In the AECC integration **Configure** dialog, set **Max charge power** and **Max discharge power**. The package reads these limits from the setpoint entity, so they are not set in the YAML.
5. Copy [`aecc_battery_to_m1.yaml`](aecc_battery_to_m1.yaml) into `/config/packages/` (same folder as HBC's `house_battery_control*.yaml`).
6. Do **not** also load a Marstek Venus M1 package (Modbus or otherwise) — the entity IDs collide.
7. Restart Home Assistant (or reload *All YAML configuration* in Developer tools).
8. Check that the template entities were created with the intended entity IDs (`sensor.marstek_m1_*`, `number.marstek_m1_*`, `select.marstek_m1_*`). `default_entity_id` is only applied the first time an entity is created.
9. Set the two helpers `AECC Battery Capacity (kWh)` and `AECC Battery Nominal Voltage (V)` to match your unit.

## Configuration

Home Assistant does not allow variables in entity IDs, so the package uses exactly two prefixes everywhere:

| Prefix | Meaning | Change it when |
| --- | --- | --- |
| `marstek_m1` | The HBC battery slot (fake Marstek entities and helpers). | You want another slot: `marstek_m2`, `marstek_m3`, … |
| `aecc_battery` | The AECC integration entities (derived from the device friendly name). | You gave the device a different friendly name, e.g. `aecc_battery_2`. |

Generate a second package for slot M2 driving a device called "AECC Battery 2":

```sh
sed 's/marstek_m1/marstek_m2/g; s/aecc_battery/aecc_battery_2/g' aecc_battery_to_m1.yaml > aecc_battery_2_to_m2.yaml
```

Every unique ID, helper key, utility meter key and the automation ID contains one of the two prefixes, so the result is a complete, non-conflicting second package.

Optional settings live in the automation's **Settings** block:

| Variable | Default | Purpose |
| --- | --- | --- |
| `RELEASE_TO_SELF_CONSUMPTION` | `false` | When HBC releases control (control mode `disable`), switch the battery to *Self-Consumption (AI)* instead of holding it at 0 W. |
| `NOTIFY_ENABLED` / `NOTIFY_SERVICE` | `false` | Send a notification on every setpoint change. |

## How control works

| HBC action | Voltdeer (AECC local TCP) |
| --- | --- |
| `select.marstek_m1_rs485_control_mode` = `enable` | Stored in helper; automation starts applying setpoints |
| `select.marstek_m1_rs485_control_mode` = `disable` | Setpoint `0` (or *Self-Consumption (AI)* if `RELEASE_TO_SELF_CONSUMPTION: true`) |
| Forcible `charge` + charge power `P` | `number.aecc_battery_power_setpoint` = `+P` |
| Forcible `discharge` + discharge power `P` | `number.aecc_battery_power_setpoint` = `−P` |
| Forcible `stop` | `number.aecc_battery_power_setpoint` = `0` |
| `select.marstek_m1_user_work_mode` | Reads `select.aecc_battery_work_mode` (`manual` ↔ *Custom / Manual*, `anti-feed` / `ai` ↔ *Self-Consumption (AI)*). `anti-feed` / `ai` are always forwarded; `manual` is forwarded **only while RS485 control mode is `enable`** (see Troubleshooting). |

HBC writes its decisions to the fake Marstek entities, which store them in helpers. The automation (mode `restart`, 500 ms coalesce for rapid HBC updates) reads the helpers and writes **one signed setpoint**: positive = charge, negative = discharge, 0 = idle. Writing the setpoint automatically puts the Voltdeer in *Custom / Manual* mode. Setpoints are clamped to the max charge / discharge power configured in the AECC integration and only written when they differ from the current value.

`sensor.marstek_m1_battery_power` follows the **HBC/Marstek sign** (charge positive, discharge negative), which matches the AECC *Battery Power* sign. `sensor.marstek_m1_ac_power` (dashboard **M1 Power**) is the same value inverted (discharge positive, charge negative), as HBC expects.

`sensor.marstek_m1_inverter_state` is derived from battery power with a small dead band (−5 … +10 W = Standby); an unavailable AECC sensor maps to `Fault`.

## Power limits

- **Max charge / discharge power** are read from the `min` / `max` attributes of `number.aecc_battery_power_setpoint`, i.e. the values you set in the AECC integration *Configure* dialog. Change them there, not in the YAML.
- The AECC app has an **On Grid Output** setting (factory default **800 W**) that caps inverter output and is **not** reachable over local TCP. Raise it in the app if you want to discharge above 800 W (only on a suitable dedicated circuit).
- The AECC HA integration also needs to be configured separately to go above 800 W (a somewhat hidden option during installation; change it afterwards via *Configure*).

## Limitations

- **P1 refresh interval: 5 s minimum.** The Voltdeer SR battery cannot process setpoint changes faster than about once every 5 seconds, so the P1 meter must not be polled faster than every **5 s** (the HomeWizard P1 integration default). A faster P1 interval makes HBC send setpoints the battery cannot keep up with, resulting in lagging or missed setpoints. Expect the battery to react roughly 5 s after each HBC decision; this is normal for this hardware.
- If a HomeWizard P1 Wi-Fi meter is used, you can adjust the refresh time to **6 seconds** if needed, using [`p1_meter_power_hw.yaml`](https://github.com/fonske/marstek-venus-rs485-node-red/blob/main/home%20assistant/other-batteries/AECC-battery/with%20aecc%20integration/p1_meter_power_hw.yaml) (a REST sensor reading the meter's local API) instead of the HomeWizard integration's `p1_meter_power` value.
- **Check that the value HBC reads is really refreshed every 6 s.** HBC (Node-RED) reads the entity `sensor.p1_meter_power`. If the HomeWizard integration already owns that entity ID, the new REST sensor gets `sensor.p1_meter_power_2` instead, and HBC silently keeps using the 5 s HomeWizard value. Open Developer tools → States, select `sensor.p1_meter_power` and watch *Last updated*: it must change every **6 s**. If it changes every 5 s, or the REST sensor shows up as `sensor.p1_meter_power_2`, fix the IDs in Home Assistant: Settings → Devices & services → Entities → open the HomeWizard entity → rename its Entity ID to `sensor.p1_meter_power_2`, then open the REST sensor and set its Entity ID to `sensor.p1_meter_power` (alternatively, point HBC's P1 sensor setting to the REST sensor's ID). Restart or reload, then check *Last updated* again.

## Safety

- Helper / number **max** is **2400 W**; the AECC integration limits still clamp all writes.
- Start with modest limits (e.g. **800 W**) and raise them when you have verified behaviour.
- Only run local TCP control on a trusted private network.
- You are responsible for safe operation; keep the AECC app available to take back control.

## Troubleshooting

- **HBC dashboard does not show the battery:** `sensor.marstek_m1_device_name` must not be `unknown`. Check that the template entities got the intended entity IDs (step 8 above).
- **Entities exist but read 0 / unavailable:** verify the AECC integration entities (`sensor.aecc_battery_battery_soc`, `sensor.aecc_battery_battery_power`) update. If your device has a different friendly name, the `aecc_battery` prefix must be replaced.
- **Setpoint is written but the battery does not follow:** check the *On Grid Output* cap in the AECC app and the max charge / discharge power in the integration.
- **Self-consumption enables control but never charges / discharges:** check the HBC PID gains (`Kp` / `Ki` / `Kd`). If all are `0`, HBC keeps forcible mode at `stop` @ `0 W`. Pick a PID preset on the HBC dashboard.
- **Battery keeps falling back from Self-Consumption (AI) to Custom / Manual, logbook says "triggered by Supervisor":** the HBC master-switch flow in Node-RED (which reaches HA through the Supervisor proxy, hence the attribution) re-applies user work mode `manual` **every 5 minutes** while the HBC master mode is **Manual control**. Since v1.1 the package ignores `manual` unless HBC has control (RS485 control mode `enable`), so the battery keeps its own mode. If you want the battery to run its own AI, use HBC master mode **Marstek control**, which never writes the work mode. In **Full control** the battery is always in Custom / Manual because every setpoint write puts it there — that is by design.
- **Battery stays in Custom / Manual after HBC releases control:** that is by design (setpoint 0). Set `RELEASE_TO_SELF_CONSUMPTION: true` in the automation to hand it back to *Self-Consumption (AI)*.
- **Enable notifications** (`NOTIFY_ENABLED: true`) to see each HBC decision and the resulting setpoint.

## Hardware validation checklist

Use Developer Tools → States / Actions before enabling full HBC strategies.

### Reads

- [ ] `sensor.marstek_m1_battery_state_of_charge` follows `sensor.aecc_battery_battery_soc`
- [ ] `sensor.marstek_m1_battery_power` shows signed W (charge +, discharge −)
- [ ] `sensor.marstek_m1_ac_power` is the inverse (dashboard **M1 Power**)
- [ ] `sensor.marstek_m1_battery_total_energy` equals the capacity helper
- [ ] `sensor.marstek_m1_battery_remaining_capacity` = SOC × capacity
- [ ] `sensor.marstek_m1_device_name` shows `Voltdeer SR5000 Pro`
- [ ] `sensor.marstek_m1_inverter_state` maps Standby / Charge / Discharge
- [ ] `number.marstek_m1_max_charge_power` / `..._max_discharge_power` show the AECC integration limits

### Writes (manual; start below device rating if preferred)

- [ ] Set `select.marstek_m1_rs485_control_mode` → `enable`
- [ ] Set forcible mode `charge` + charge power e.g. `500` → `number.aecc_battery_power_setpoint` = `500`, battery charges
- [ ] Set forcible mode `discharge` + discharge power e.g. `500` → setpoint = `−500`, battery discharges
- [ ] Set forcible mode `stop` → setpoint `0`, Standby
- [ ] Set RS485 control → `disable` → setpoint `0` (or *Self-Consumption (AI)* when enabled)

### HBC strategies

- [ ] HBC **Charge** drives a positive setpoint
- [ ] HBC **Sell** drives a negative setpoint
- [ ] HBC **Self-consumption** (PID) updates the setpoint continuously
- [ ] Leaving Full Control returns the unit to idle / self-consumption

## Multi-battery

Duplicate the file for `m2` / `m3` with the `sed` command in [Configuration](#configuration): replace `marstek_m1` with the new slot and `aecc_battery` with the entity prefix of the second device.
