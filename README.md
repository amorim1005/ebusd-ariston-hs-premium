# ebusd config for Ariston HS Premium 24 (BridgeNet) + Home Assistant

An [ebusd](https://github.com/john30/ebusd) configuration for the **Ariston HS Premium 24 EU2** gas combi
boiler, which talks Ariston's **BridgeNet®** dialect of eBUS, plus a step-by-step guide to run it with the
ebusd add-on in Home Assistant.

Every register in this repository was **verified on a real boiler**: passive bus captures, read-only
requests, and tests with heating and hot-water draws. Nothing here was copied blindly from other models.

> ### ⚠️ Read-only by design
> This configuration **never writes to the boiler**: it has no `w` lines. Please keep it that way unless
> you really know what you are doing. A single accidental write to a BridgeNet setting (for example
> setting a zone's thermoregulation mode to "outdoor + room probe") can leave the boiler in a permanent
> error (411/412 "room probe missing") that only goes away by changing the setting on the boiler or
> fitting a probe. Do not run ebusd with `--enablehex`, and do not add write definitions you have not
> tested.

## What you get in Home Assistant

Passive (broadcast by the boiler, no bus traffic from ebusd):

| Entity | Meaning |
|---|---|
| `status boiler_status` | standby / heating / dhw / circulating / low_water_pressure / no_flame / … |
| `status ta_contact` | room thermostat contact (TA1) on/off |
| `heat_request` | zone 1 heat request |
| `flow_setpoints boiler`, `flow_setpoints z1` | computed flow setpoint (°C) |
| `heating_status`, `dhw_status` | heating (winter) / DHW enabled on the panel |
| `dhw_setpoint` | DHW comfort setpoint (°C), updated as soon as it is changed on the panel |
| `dhw_comfort_mode` | off / timed / always_on |
| `z1_heat_setpoint` | heating setpoint set on the panel (°C) |
| `z1_flow_setpoint_limits max/min`, `temp_settings antifreeze/z1_day`, `z1_night_temp`, `z1_thermoreg_mode` | zone 1 settings |

Active reads (ebusd polls the boiler with read-only requests):

| Entity | Meaning |
|---|---|
| `flow_temp`, `return_temp` | flow / return temperature (°C) |
| `flame` | flame on/off |
| `burner_power` | burner power in kW (0 when off, up to ~25 kW) |
| `burner_modulation` | burner modulation in % (about 2 when off) |
| `fan_speed` | fan speed (rpm) |
| `pump_modulation` | circulation pump modulation (%) |
| `burner_request` | idle / active (burner demand, heating or DHW) |
| `dhw_flow` | DHW flow while a tap is open (0 otherwise; scale not yet calibrated) |
| `hours_ch`, `hours_dhw`, `ignitions`, `pump_hours`, `diverter_cycles`, `circulation_cycles`, `fan_cycles` | counters |

Not available on this model: **water pressure** (the HS Premium 24 has a pressure *switch* and an analogue
gauge, no pressure sensor; low pressure shows up as `boiler_status = low_water_pressure`) and the
**outdoor temperature** (only if a correct NTC probe is fitted).

See [docs/registers.md](docs/registers.md) for the full register list and the protocol notes.

## Requirements

- An eBUS adapter with an *enhanced* TCP port, e.g. the esp-eBus adapter from
  [danielkucera/esp-ebus](https://github.com/danielkucera/esp-ebus) (port 3335 = enhanced, 3334 = listen only).
- Home Assistant with an MQTT broker (e.g. the Mosquitto add-on) and an **ebusd add-on**
  (tested with ebusd 26.1).
- The **File editor** add-on (or any other way to put a file in `/share`).

## Installation

### 1. Put the CSV in `/share/ebusd`

The ebusd add-on cannot see your Home Assistant config folder (`/homeassistant`). It only sees its own
folder (`/config` inside the add-on, which is `/addon_configs/<slug>/` on the host) and **`/share`**.
The easiest place is `/share/ebusd`.

1. In the **File editor** add-on, open its **Configuration** tab and turn **off** "Enforce Basepath".
   Save and restart the File editor. You can now browse outside `/homeassistant`.
2. In the File editor, go up to the root, open `share`, create the folder `ebusd`.
3. Create the file `ariston-hs24.csv` in `share/ebusd/` and paste the content of
   [`ariston-hs24.csv`](ariston-hs24.csv). The **first line must be the `# type (r[1-9];w;u),…` header**.

### 2. Configure the ebusd add-on

- **Network device**: `enh:<adapter-ip>:3335`
- **Additional ebusd options** (one per entry):

  ```
  --mqttjson
  --configpath=/share/ebusd
  --latency=50
  --receivetimeout=100
  --sendretries=3
  --pollinterval=10
  --mqttvar=filter-name=status|temp|setpoint|request|flame|modulation|fan|valve|hours|ignition|power|flow|pump|cycles|mode
  ```

- Do **not** add `--enablehex` or `--scanconfig`.
- Optional: enable the add-on's SQLite telegram log (e.g. `/share/ebusd/telegrams.db`). It is very handy
  for debugging.

Start the add-on. In its log you should see:

```
found messages: 28 (0 conditional on 0 conditions, 16 poll, 12 update)
received update-read boiler status: standby;off
sent poll-read boiler flow_temp QQ=31: 45.2
```

The device **"ebusd boiler"** appears under *Settings → Devices & services → MQTT*. Entities appear as
soon as ebusd has a value for them: the settings (`2020`) can take up to 30 minutes (or change a setpoint
on the boiler panel), and the counters take a few polling rounds.

### Why these options

- `--mqttvar=filter-name=…`: the bundled `mqtt-hassio.cfg` only publishes messages whose *name* matches
  `status|temp|…|hours|…`. Without this override, entities such as `burner_power`, `flame` or `heat_request`
  never show up.
- `--latency=50 --receivetimeout=100`: Wi-Fi adapters add latency; with the defaults many polls fail with
  `wrong symbol received` / `SYN received`. **`--receivetimeout` must not exceed 100** (ebusd 26.1 refuses to
  start with `invalid receivetimeout`).
- `--pollinterval=10`: halves the bus load. Values still refresh roughly once a minute.
- `--configpath` pointing to a local folder is mandatory when ebusd does not scan, otherwise it stops with
  `invalid configpath without scanconfig`.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `--configpath=/homeassistant/ebusd does not exist` | the add-on cannot see `/homeassistant`; use `/share/ebusd` |
| `missing name in column 9` | the header line is missing (ebusd then assumes the newer column layout), or an `IGN` field has no name |
| `invalid receivetimeout` | use `--receivetimeout=100` or less |
| Entities missing in HA | check the `filter-name` option; restart the add-on once all messages have been read |
| Old entities stay "unavailable" | ebusd publishes MQTT discovery with *retain*: publish an empty retained payload to the entity's discovery topic (*MQTT → Configure → Publish a packet*), shown under *MQTT Info* |
| Many `poll … failed` lines | Wi-Fi latency: improve the adapter's signal; the values still update on the next round |

## Multiple zones

The boiler firmware always broadcasts data for 6 zones (`x1`..`x6` in the register IDs), even with a single
zone installed. This config only exposes zone 1. If you have a zone manager, the other zones are in the same
telegrams (see [docs/registers.md](docs/registers.md)): turn the corresponding `IGN` bytes into named fields.

## Credits

Built on the work of the BridgeNet community:
[ysard/ebusd_configuration_chaffoteaux_bridgenet](https://github.com/ysard/ebusd_configuration_chaffoteaux_bridgenet),
[wrongisthenewright/ebusd-configuration-ariston-bridgenet](https://github.com/wrongisthenewright/ebusd-configuration-ariston-bridgenet),
[komw/ariston-bus-bridgenet-ebusd](https://github.com/komw/ariston-bus-bridgenet-ebusd),
the [Home Assistant forum thread](https://community.home-assistant.io/t/ariston-group-integration-via-ebusd/738887),
and of course [ebusd](https://github.com/john30/ebusd) and [esp-ebus](https://github.com/danielkucera/esp-ebus).

## Disclaimer

Use at your own risk. This is not affiliated with Ariston. Read-only access is low risk, but you are connecting
equipment to your heating system: follow the adapter's wiring and power instructions.
