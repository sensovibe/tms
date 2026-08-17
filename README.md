# SANITRAC TMS — Toilet Monitoring System

Firmware and documentation for the SANITRAC Toilet Monitoring System (TMS‑V1.2), by Sensovibe.

**Deployment target:** Nashik Trimbakeshwar Simhastha Kumbh 2027 — mass‑gathering public toilets.

## Hardware (Top Module)

| Block | Part |
|---|---|
| MCU | ESP32‑S3 (WROOM‑1) |
| Gas sensing | MQ‑2, MQ‑4, MQ‑9, MQ‑137 |
| Environment | Temperature & humidity sensor *(part TBC)* |
| Identity | RFID reader — cleaner attendance |
| Occupancy | Person‑count sensor, IR + motion headers, door/reed switch |
| Actuation | Relay card → motor pump, solenoid valve |
| Feedback | Indicator |
| Power | Hylink AC‑DC module *(part TBC)* |

## Documentation

- [**Gas sensor research**](docs/gas-sensors-research.md) — what each sensor detects, what values it should
  show, thresholds, calibration procedure, the odour‑index algorithm, and open hardware questions.
- [**Sensor calibration reference**](docs/sensor-calibration.json) — the same constants in machine‑readable
  form, for the firmware to consume.

## Status

Research and specification phase. No firmware yet — see
[§10 of the gas sensor research](docs/gas-sensors-research.md#10-what-i-need-to-write-the-firmware)
for the information still needed (schematic, pin map, divider values, MQ‑9 heater drive).
