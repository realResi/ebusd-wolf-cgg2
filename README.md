# ebusd-wolf-cgg2

ebusd configuration for the **Wolf CGG-2** gas condensing boiler with **Wolf BM** control module, developed through live eBUS reverse engineering.

This repository provides a complete, ready-to-use stack including MQTT auto-discovery for Home Assistant.

> **⚠️ Disclaimer:** This configuration is provided as-is, based on findings from a single CGG-2-18 installation. Use at your own risk. Incorrect parameter settings may affect boiler operation. Always verify critical parameters against your device documentation before writing — especially:
> - **HG21** (min. boiler temperature): Do not set below 40°C — corrosion protection for steel heat exchanger. The BM display may show a wider range (20–90°C), but 40°C is the minimum recommended by Wolf for the CGG-2.
> - **HG22** (max. boiler temperature): Observe manufacturer limits.
> - **HG23** (DHW max. temperature): Risk of scalding. Verify safe limits for your installation before writing.

---

## Tested with

- Wolf CGG-2-18 (Feuerungsautomat: Kromschröder, SW=0204)
- Wolf BM control module (Master=0x30, Slave=0x35, SW=0204)
- Wolf Funk-Empfänger wireless receiver (Master=0x0f, Slave=0x0a)
- ebusd v26.1.8 (Home Assistant add-on)
- Home Assistant 2026.5.3 with MQTT (Mosquitto) integration

---

## Bus Layout

```
QQ=03 / ZZ=08  — Feuerungsautomat (boiler controller, Kromschröder)
QQ=30 / ZZ=35  — BM control module (master/slave)
QQ=0f / ZZ=0a  — Funk-Empfänger (wireless receiver, master/slave)
QQ=f1          — Third-party room thermostat (e.g. tado° via E1 relay, HG13=1)
```

Bus scan result:
```
address 08: slave  MF=Kromschroeder  SW=0204  (Feuerungsautomat)
address 0a: slave  MF=Kromschroeder  SW=0204  (Funk-Empfänger slave)
address 35: slave  MF=Kromschroeder  SW=0204  (BM slave)
```

---

## Files

| File | Description |
|---|---|
| `15.csv` | Main config: HG parameters, operating data, error history, eBUS analysis |
| `_templates.csv` | Field type templates (standard Wolf/Kromschröder) |
| `broadcast.csv` | Broadcast messages incl. outside temperature from Funk-Empfänger |
| `mqtt-hassio.cfg` | MQTT auto-discovery for Home Assistant |

---

## File Placement

Place all files in your ebusd configuration directory. With the ebusd Home Assistant add-on, this is typically `/addon_configs/<addon_id>/` (mapped to `/config/` inside the add-on container).

Set the ebusd option:
```
--configpath=/config
```

> **Important:** Do **not** use `--scanconfig` — the CGG-2 Feuerungsautomat (ZZ=08) returns an empty ID string which prevents automatic scanconfig matching. Use `--configpath` with direct file loading instead.

Directory layout:
```
/config/
├── 15.csv
├── _templates.csv
├── broadcast.csv
└── mqtt-hassio.cfg
```

### Why `15.csv`?

The filename follows the ebusd scanconfig convention where filenames correspond to the slave address of the device. `0x15` (decimal 21) is the slave address of the second controller (Regler 2) in the Kromschröder/Wolf bus layout. This enables direct use with `--scanconfig` if device identification succeeds in future ebusd versions.

---

## Working Parameters

### Operating Data (circuit: feuerung, ZZ=08, PBSB=5022)

| Name | Register | Description |
|---|---|---|
| `temp_burner` | CC0D00 | Boiler temperature °C |
| `performance_burner` | CC6F01 | Modulation % |
| `performance_pump` | CC5727 | Pump speed % |
| `no_of_firing` | CC2602 | Burner starts (total) |
| `op_hrs_heating` | CC2A02 | Burner operating hours |
| `pressure` | CC1A27 | System pressure bar |
| `vorlauf_soll` | b80200 | Flow setpoint °C |
| `vorlauf_ist` | 280d00 | Flow actual °C |
| `warmwasser_soll` | e40300 | DHW setpoint °C |
| `warmwasser_ist` | cc0e00 | DHW actual °C |

### HG Parameters (circuit: feuerung, ZZ=08, PBSB=5022/5023)

All HG parameters support read (`r`) and write (`w`). Write uses PBSB=5023 with suffix `9d010000`.

| Parameter | Description | Range |
|---|---|---|
| HG01 | Burner hysteresis | 5–25 K |
| HG06 | Pump mode | 0–2 |
| HG07 | Pump run-on time | 0–30 min |
| HG08 | Max flow temperature | 40–90 °C |
| HG09 | Burner lockout time | 1–30 min |
| HG10 | eBUS address | — |
| HG11 | DHW quick start | 10–60 °C |
| HG12 | Gas type | — |
| HG13 | Input E1 function | 0–11 (Wolf manual: 1–10; 0 and 11 confirmed working) |
| HG15 | DHW hysteresis | 1–30 K |
| HG16 | Min pump speed HC | 20–100 % |
| HG17 | Max pump speed HC | 20–100 % |
| HG21 | Min boiler temp (corrosion protection) | 20–60 °C (**>40°C strongly recommended**) |
| HG22 | Max boiler temp | °C |
| HG74 | Fan speed | U/sec |
| HG75 | DHW flow rate | l/min |
| HG90 | Burner operating hours | h |
| HG91 | Burner starts | — |

### Error History (HG80–HG89)

Ten error history registers. Raw 2-byte values — encoding is proprietary and not fully decoded. Stored as HEX for reference.

---

## Known Limitations

- **`temp_return`** (return temperature): No sensor fitted in my CGG-2 hardware — register exists but always returns 0x7FFF. Entry commented out in `15.csv`.
- **PBSB=0503 / PBSB=0800**: `b`-type passive matching does not work in ebusd v26 for these message types. Documented in `15.csv` for reference.
- **HG23 write**: Must be sent as BM broadcast (ZZ=fe), not as direct MS write. See BM Broadcast section below.

---

## BM Broadcast Messages (QQ=30, ZZ=fe, PBSB=5022)

The BM control module broadcasts its own A-parameters periodically. These can be decoded passively but are not writable via standard MS commands.

**Frame structure:** `30 fe 5022 09 [ID 3 bytes] [value SIN16LE/10] 5d010000`

**ID schema:** `[CRC prefix byte, ignored by device] [TelegramNr LE 2 bytes]`

All IDs are readable via: `hex -s 31 35 5022 03 XX XX XX` (ZZ=35 = BM slave)

| ID | TelegramNr | Parameter | Example value |
|---|---|---|---|
| 248101 / 6c8101 | 385 | A14 = HG23 (DHW max temp) | 65.0°C |
| 440f01 | 271 | A00 (room influence factor) | 4 |
| 28000a | 2560 | A09 (frost protection) | -10.0°C |
| 08be02 | 702 | A12 (setback stop) | -20.0°C |
| acbd02 | 701 | A11 (summer/winter switchover) | 0=off |
| 241401 | 276 | unknown | — |
| 642300 | 35 | unknown | — |

TelegramNr mapping source: `ism7mqtt` ParameterTemplates.xml (zivillian/ism7mqtt)

### Writing HG23 via broadcast

```bash
# Example: set HG23 to 65°C (650 × 10 = 0x028a → LE: 8a 02)
echo "hex -s 30 fe502309248101{lo}{hi}5d010000" | nc 172.30.33.1 8888
```

---

## Home Assistant Integration

### MQTT auto-discovery

Use `mqtt-hassio.cfg` with the ebusd `--mqttint` option:

```
--mqttjson --mqttint=/config/mqtt-hassio.cfg
```

Sensors appear automatically in Home Assistant via MQTT Discovery.

### Polling automation

Several registers require active polling (`read -f`). Example shell commands for HA automation:

```yaml
shell_command:
  poll_vorlauf_ist: "echo 'read -f -c feuerung vorlauf_ist' | nc 172.30.33.1 8888"
  poll_warmwasser_ist: "echo 'read -f -c feuerung warmwasser_ist' | nc 172.30.33.1 8888"
  poll_performance_burner: "echo 'read -f performance_burner' | nc 172.30.33.1 8888"
```

### DHW one-time heating (1xWW)

Triggering a DHW heating cycle outside the scheduled times:

1. Read and back up current HG15 value
2. Set HG15 temporarily to 1 K → boiler ignites
3. Monitor DHW actual temperature
4. Restore original HG15 value after target temperature is reached

> **Note:** This only works during DHW heating time windows configured in the BM. Outside these windows, the FA ignores the setpoint change.

---

## BM Clock Correction

The Wolf BM has no battery-backed RTC — after a power cut it loses its time and date.

With ebusd, the clock can be corrected automatically. The BM accepts datetime broadcasts from `QQ=0f` (Funk-Empfänger) or `QQ=30` (BM master) only. ebusd allows spoofing the source address via `hex -s` on TCP port 8888:

```bash
bcd() { printf "%02x" $(( (($1/10)<<4) | ($1%10) )); }
echo "hex -s 0f fe 07 00 09 80 00 $(bcd $(date +%-S)) $(bcd $(date +%-M)) $(bcd $(date +%-H)) $(bcd $(date +%-d)) $(bcd $(date +%-m)) $(date +%u) $(bcd $(date +%y))" | nc -w3 172.30.33.1 8888
```

This can be automated in Home Assistant — triggered on RTC reset detection, time drift, or daily as a preventive measure.

---

## Acknowledgements

This configuration was developed by reverse engineering the Wolf CGG-2 eBUS protocol through live bus analysis, parameter correlation, and cross-referencing with Wolf SmartSet XML data.

Development was carried out with significant assistance from Claude (Anthropic AI), which supported protocol analysis, decoding, and documentation.

Community resources that contributed:
- [ebusd](https://github.com/john30/ebusd) — the daemon this configuration targets
- [ebusd Discussion #779](https://github.com/john30/ebusd/discussions/779) — Wolf hardware collection
- [ism7mqtt](https://github.com/zivillian/ism7mqtt) — Wolf SmartSet parameter extraction (TelegramNr mapping)
- [puni2k/ebusd-configuration-wolf](https://github.com/puni2k/ebusd-configuration-wolf) — Wolf ebusd config collection (CGG-2 config merged in PR #12)
- FHEM forum topic 95173 (pink99panther) — source for `_templates.csv` field type definitions

---

## License

MIT License — see [LICENSE](LICENSE) file.
