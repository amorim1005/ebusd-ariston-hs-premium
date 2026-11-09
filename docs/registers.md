# Ariston HS Premium 24 (BridgeNet) – register notes

Everything below was observed on one Ariston HS Premium 24 EU2 (boiler at eBUS master address `37`,
slave `3c`, with no eBUS thermostat: heat demand comes from a dry contact on TA1). Confidence:
**✅ confirmed** (seen changing as expected in tests), **🟡 likely**, **❔ unknown / not useful**.

## Protocol basics

BridgeNet uses Ariston-specific PBSB codes. All data are lists of **records**:

```
<ID: 2 bytes> <value: 1 or 2 bytes> [<status byte>]
```

The **value size depends only on the first ID byte**. This held for all 562 IDs that exist on this boiler:

| 1st ID byte | value | typical use |
|---|---|---|
| `00`–`1f` | 1 byte | on/off |
| `20`–`3f` | 2 bytes | (rare) |
| `40`–`5f` | 2 bytes, unsigned | counters (UIN) |
| `60`–`7f` | 2 bytes, signed /10 | temperatures, kW (SIN, `7fff` = no value) |
| `a0`–`bf` | 1 byte | (rare) |
| `c0`–`df` | 1 byte, unsigned | states, % (UCH) |
| `80`–`9f`, `e0`–`ff` | – | no IDs found |

Message types seen on this boiler:

| PBSB | Direction | Content |
|---|---|---|
| `2010` | `37 → FE` broadcast | live state, records **with** a status byte (always `00` so far, `ff` seen on an invalid outdoor probe). Sent every ~5–25 s. |
| `2020` | `37 → FE` broadcast | settings, records **without** a status byte. Full dump every ~30 min, and immediately when a setting is changed on the panel. |
| `203b` | `37 → FE` broadcast | 1 byte (`3c`), every ~10 min |
| `2000` | `31 → 3c` request / reply | read: request `02 <ID>`, reply `01 <value>`; reply `00` = ID does not exist |

Because records in a broadcast always come in the same order, an ebusd `u` line uses the first record's ID as
the message ID and covers the other bytes with named `IGN` fields (every field needs a name).

## `2010` – live state (passive)

| ID | Meaning | Type | Conf. |
|---|---|---|---|
| `c04b` | boiler status: 1 standby, 2 heating, 3 DHW, 5 circulating, 51 low water pressure, 52 no flame, 65 rebooting | UCH | ✅ |
| `0919` | room thermostat contact TA1 | 1 byte | ✅ |
| `0191`..`0691` | heat request zone 1..6 | 1 byte | ✅ (z1) |
| `6047` | boiler flow setpoint (computed) | SIN/10 | ✅ |
| `6197`..`6697` | computed flow setpoint zone 1..6 | SIN/10 | ✅ (z1) |
| `6996`..`6e96` | zone 1..6 value 19.0 (room/day setpoint) | SIN/10 | 🟡 |
| `7310`, `7647` | outdoor temperature (only meaningful with a correct NTC probe) | SIN/10 | 🟡 |
| `6147` | constant 8.0 here (not the DHW temperature on this model) | SIN/10 | ❔ |
| `d746` | DHW comfort status (0 here) | UCH | ❔ |
| `c194`.., `d346`, `d742`, `d994`.., `da46`/`dc46`/`de46`, `d1f2`.. | not identified | UCH | ❔ |

## `2020` – settings (passive)

| ID | Meaning | Conf. |
|---|---|---|
| `0120` / `0220` | heating (winter) / DHW enabled | ✅ |
| `6126` | DHW comfort setpoint | ✅ (follows the panel immediately) |
| `6226` | DHW economy setpoint (10.0 here) | 🟡 |
| `c028` | DHW comfort mode: 0 off, 1 timed, 2 always on | 🟡 |
| `6571`..`6576` | heating setpoint (fixed temperature), zones 1..6 | ✅ (z1) |
| `6071` / `6171` | max / min flow temperature, zone 1 (`x2..x6` for other zones) | ✅ |
| `6271` / `6371` | day / night temperature, zone 1 | ✅ |
| `6471` / `6a71` | thermoregulation offset / slope, zone 1 | 🟡 |
| `c079`..`c07e` | thermoregulation type, zones 1..6: 0 fixed, 1 on/off, 2 room only, 3 outdoor only, 4 outdoor + room | ✅ |
| `c279` | room temperature influence | 🟡 |
| `6426` | antifreeze temperature | ✅ |
| `0081`..`0086` | heating temperature range per zone | 🟡 |
| `0f23` | cooling enabled | 🟡 |
| `7426` | outdoor temperature offset | 🟡 |

> Setting `c07x` to 4 (outdoor + room) without a room probe causes error 411/412.
> **Never write these registers unless you know what you are doing.**

## `2000` – read requests (active)

| ID | Meaning | Type | Conf. |
|---|---|---|---|
| `6810` / `6910` | flow / return temperature | SIN/10 °C | ✅ |
| `0e11` | flame | 1 byte | ✅ |
| `6847` | burner power, 0 when off, ~8–25 kW when burning | /10 kW | ✅ |
| `4013` | fan speed (≈ 240 rpm per kW) | UIN rpm | ✅ |
| `c314` | burner modulation, ~95 % at start, about 2 when off | UCH % | ✅ |
| `c300` | circulation pump modulation | UCH % | ✅ |
| `c000` | burner demand: 0 idle, 2 active (heating **and** DHW) | UCH | ✅ |
| `61a1` | DHW flow while a tap is open (≈10 low, ≈18 full flow; scale not calibrated) | SIN/10 | ✅ (relative) |
| `62a1` | constant 55.6 while DHW is drawn, 0 otherwise | SIN/10 | ❔ |
| `7da0` / `71a0` | internal DHW targets (follow the DHW setpoint, not 1:1) | SIN/10 | 🟡 |
| `658a`, `68a0`, `7c2b` | equal the heating setpoint | SIN/10 | 🟡 |
| `4bd1` / `4cd1` | burner hours heating / DHW | UIN h | ✅ |
| `42d8` | ignitions | UIN | ✅ |
| `42d2` | pump hours | UIN h | 🟡 |
| `40d2` / `44d2` / `4cd2` | diverter / circulation / fan cycles | UIN | 🟡 |
| `c328` | pressure monitoring device: 1 = pressure switch | UCH | ✅ |
| `7547` | water pressure on models with a sensor; always 0 here | – | n/a |
| `c404`, `c012` | always 0 here (not modulation / not diverter on this model) | – | ❔ |
| `6147` | not readable (`00`) | – | n/a |

A full read-only scan of all 65 536 IDs (`31 → 3c 2000`) found 562 existing IDs on this boiler; many are
counters and statistics (`40ab`..`5fab`, `40ac`..`5cac`, `40d2`..`51d2`, `42d9`..`5bd9`), some forming
32-bit values from two UIN halves.
