# ESPHome Growatt Bridge (ESP32-C3)

An [ESPHome](https://esphome.io/) configuration that turns a cheap **ESP32-C3** board into a Modbus bridge for a single-phase **Growatt inverter** (MIC / MID TL-X series). It reads live data from the inverter over RS485/Modbus RTU, exposes it to **Home Assistant**, serves a built-in **web dashboard**, and re-publishes the data as a **SunSpec Modbus TCP server** so it can be picked up by **Victron** equipment (e.g. GX devices / VenusOS) — including support for Victron's **Dynamic Power Reduction**.

Version 1.1 implemented a "Volt Watt mode" algorith that reduced active power when the grid voltage becomes too high.
As a bonus, it also supports old-school **RRCR / DRM (Demand Response Mode)** power curtailment via 4 relays, for inverters/setups where the ripple-control input is wired up.

> Credit to the projects this build stands on:
> - [JasperE84/Growatt_ESPHome_ESP32_Modbus_RS485_Example](https://github.com/JasperE84/Growatt_ESPHome_ESP32_Modbus_RS485_Example)
> - [mahoekst/SunspecModbusServer](https://github.com/mahoekst/SunspecModbusServer/tree/main)
> - [Tweakers.net discussion thread](https://gathering.tweakers.net/forum/list_messages/2153412)

---

## Table of Contents

- [What This Does](#what-this-does)
- [Hardware Requirements](#hardware-requirements)
- [Wiring / Pinout](#wiring--pinout)
- [Software Prerequisites](#software-prerequisites)
- [Configuration](#configuration)
  - [secrets.yaml](#secretsyaml)
  - [Substitutions](#substitutions)
- [Flashing](#flashing)
- [Using the Bridge](#using-the-bridge)
  - [Web Dashboard](#web-dashboard)
  - [Home Assistant](#home-assistant)
  - [SunSpec / Victron Integration](#sunspec--victron-integration)
  - [RRCR / DRM Power Control](#rrcr--drm-power-control)
  - [Status LEDs](#status-leds)
- [Sensors & Entities Reference](#sensors--entities-reference)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## What This Does

The ESP32-C3 talks to the Growatt inverter over RS485 (Modbus RTU) and:

1. **Reads** live AC/DC electrical values, daily/total production, temperature, and inverter status.
2. **Exposes** all of this as entities in Home Assistant via the native ESPHome API, plus a local **web UI** on port 80.
3. **Re-serves** the data over **Modbus TCP (port 502)** using the **SunSpec** protocol, so a Victron system can treat the Growatt inverter as a SunSpec-compatible PV inverter — including accepting power limit commands from Victron's Dynamic Power Reduction feature.
4. Optionally drives **4 relays** to implement legacy **RRCR/DRM** ripple-control power curtailment (0% / 30% / 60% / 100%), for sites where that hardware input is used instead of (or alongside) Modbus-based curtailment.

## Hardware Requirements

| Component | Notes |
|---|---|
| ESP32-C3 dev board | Configured for `esp32-c3-devkitm-1`, Arduino framework |
| RS485 transceiver module | Connected to UART TX/RX + a flow-control (DE/RE) pin |
| 2x LED (or a bi-color LED) | Status indication — red = fault/off, green = OK |
| 4x relay module (optional) | Only needed if you plan to use the RRCR/DRM power-level feature |
| Growatt single-phase inverter (MIC/MID TL-X series) | Must support Modbus RTU (RS485) |

## Wiring / Pinout

| Function | GPIO | Notes |
|---|---|---|
| UART RX | GPIO20 | From RS485 transceiver |
| UART TX | GPIO21 | To RS485 transceiver |
| RS485 flow control (DE/RE) | GPIO7 | Toggles transceiver direction |
| Red LED | GPIO8 | Active-low (inverted) |
| Green LED | GPIO5 | Active-low (inverted) |
| Relay K1 — 0% power | GPIO6 | Active-low, restores OFF on boot |
| Relay K2 — 30% power | GPIO3 | Active-low, restores OFF on boot |
| Relay K3 — 60% power | GPIO9 | Active-low, restores OFF on boot |
| Relay K4 — 100% power | GPIO10 | Active-low, restores **ON** on boot |

UART settings: `9600 baud, 8N1`. Modbus slave address of the inverter is `0x1`.

## Software Prerequisites

- [ESPHome](https://esphome.io/guides/getting_started_command_line.html) (CLI or Home Assistant add-on)
- A `secrets.yaml` file alongside this YAML (see below)
- The **`sunspec_modbus_server`** external component, referenced locally:
  ```yaml
  external_components:
    - source:
        type: local
        path: components
      components: [ sunspec_modbus_server ]
  ```
  You need a `components/sunspec_modbus_server/` folder (from the [SunspecModbusServer](https://github.com/mahoekst/SunspecModbusServer) project) placed next to this YAML file before compiling.

## Configuration

### secrets.yaml

Create a `secrets.yaml` file in the same directory with:

```yaml
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
esphome_api_key: "base64-encoded-32-byte-key"   # generate with `esphome secrets` or an online tool
esphome_ota_password: "your-ota-password"
esphome_ap_password: "your-fallback-ap-password"
```

### Substitutions

At the top of the YAML you can adjust these to match your setup:

```yaml
substitutions:
  device_name: growatt-bridge-c3
  friendly_name: Growatt Bridge c3
  manufacturer: "Growatt"
  model: "MIC 3000TL-X"
  serial: "QUH8CKT07Q"
  version: "PV00.0038500"
  max_power: "3000"
```

- `manufacturer`, `model`, `serial`, `version`, `max_power` are cosmetic/identification values reported over SunSpec and shown as diagnostic text sensors — set them to match your actual inverter's nameplate/label for a clean Victron display.
- `max_power` is used by the SunSpec server as the inverter's rated power (Watts).

Also update the static network settings under `wifi:` to match your network:

```yaml
wifi:
  manual_ip:
    static_ip: 192.168.1.33
    gateway: 192.168.1.1
    subnet: 255.255.255.0
```

## Flashing

```bash
esphome run esphome-growatt-bridge.yaml
```

First flash requires a USB connection; subsequent updates can be done over-the-air (OTA) using the password in `secrets.yaml`. If Wi-Fi fails to connect, the device falls back to a captive-portal access point named **`Growatt-Bridge-C3`** (password from `esphome_ap_password`) so you can reconfigure it.

## Using the Bridge

### Web Dashboard

Once connected to Wi-Fi, browse to the device's IP address (e.g. `http://192.168.1.33`) for a live dashboard grouped into:

- **AC Output** — active power, voltage, frequency, daily/total production
- **DC Input (PV)** — voltage, current, power for string 1 and string 2 (if present)
- **Control** — inverter on/off switch, active power rate slider, RRCR power level selector
- **Device** — manufacturer/model/serial/version/max power, Wi-Fi signal, uptime, restart button

### Home Assistant

The device is auto-discovered by Home Assistant via the native ESPHome API (once the `esphome_api_key` matches). All sensors, the on/off switch, the power-rate slider, and the RRCR select entity become available for automations and dashboards.

### SunSpec / Victron Integration

The bridge runs a **SunSpec-compliant Modbus TCP server on port 502** (unit ID `126`), sourced from these live values:

| SunSpec field | Source entity |
|---|---|
| AC Power | `AC_active_Power` |
| AC Voltage | `AC_Voltage` |
| AC Frequency | `AC_Frequency` |
| Total Energy | `Total_Production` |
| DC Voltage | `DC1_Voltage` |
| DC Current | `DC1_Current` |
| DC Power | `DC1_Power` |
| Temperature | `Heatsink_temperature` |
| Power Limit (write target) | `Active_Power_Rate` |

In Victron/VenusOS:

1. Go to **Settings → Integrations → PV Inverters**.
2. The bridge should appear as a SunSpec device (using the `manufacturer`/`model`/`serial` you configured).
3. Select it, then enable **Dynamic Power Reduction** if you want Victron to actively curtail the inverter's output via Modbus writes to the `Active_Power_Rate` register.

### RRCR / DRM Power Control

Growatt inverters can also be curtailed via a hardware **Radio Ripple Control Receiver (RRCR)** input, following **Demand Response Mode (DRM)** signalling. This isn't enabled by default on most units — you may need to activate it via the **ShineTools app** or the inverter's local menu.

If you've wired up the 4 relays, use the **"RRCR Power level"** select entity (`0%`, `30%`, `60%`, `100%`) to switch which relay is closed. The logic guarantees only one relay is ever active at a time.

> Most users controlling power via Modbus/Victron's Dynamic Power Reduction don't need the relays at all — this is an alternative/legacy path for setups where DRM is the only supported curtailment method.

### Status LEDs

| State | LED |
|---|---|
| Inverter status = "Normal" | Green ON, Red OFF |
| Any other status (Waiting/Fault/Unknown) | Red ON, Green OFF |
| Active Power Rate < 100% | Green LED blinks — blink speed scales with how far below 100% (slow near 0%, fast near 99%) |
| Active Power Rate = 100% | Blinking stops, green LED solid |

On boot, all measurement sensors are pre-published as `0` (rather than "unavailable") so dashboards look sane while the inverter is off overnight; cumulative totals are left unset since `0` would be misleading. Power rate is initialized to 100% since that register can't be trusted while the inverter is powered down — Victron will overwrite it with the correct value once connected.

## Sensors & Entities Reference

| Entity | Type | Unit | Modbus Register(s) |
|---|---|---|---|
| Inverter On - Off | switch | — | holding reg 0, bit 0 |
| Active Power Rate | number (slider) | % | holding reg 3 |
| RRCR Power level | select | — | drives relays only (no register) |
| AC Active Power | sensor | W | read reg 35–36 |
| AC Voltage | sensor | V | read reg 38 |
| AC Frequency | sensor | Hz | read reg 37 |
| Today Production | sensor | kWh | read reg 53–54 |
| Total Production | sensor | kWh | read reg 55–56 |
| DC1 Power | sensor | W | read reg 5–6 |
| DC1 Voltage | sensor | V | read reg 3 |
| DC1 Current | sensor | A | read reg 4 |
| DC2 Power | sensor | W | read reg 9–10 |
| DC2 Voltage | sensor | V | read reg 7 |
| DC2 Current | sensor | A | read reg 8 |
| Heatsink temperature | sensor | °C | read reg 93 |
| Inverter Status | text_sensor | — | read reg 0 (Waiting/Normal/Fault/Unknown) |
| Manufacturer / Model / Serial / Version / Max Power | text_sensor | — | static, from substitutions |
| Bridge WiFi Signal | sensor | dBm | — |
| Bridge Uptime | sensor | s | — |
| Restart | button | — | — |

## Troubleshooting

- **`Received modbus data but command queue is empty` / `Stop waiting for response from 1` in logs** — expected and harmless when the inverter is off overnight; these log levels are already suppressed to `ERROR` in this config.
- **All sensors read 0 / unavailable** — normal at night when the inverter has no PV input and shuts down; values recover once the sun is up and the inverter powers on.
- **SunSpec device not found by Victron** — confirm the ESP32 and Victron GX device are on the same network/VLAN, and that nothing else is using TCP port 502 on the bridge's IP.
- **RRCR relays not switching** — the relays only respond to the "RRCR Power level" select entity; this is independent from the Modbus-based `Active Power Rate`. Make sure DRM/RRCR mode is actually enabled on the inverter itself if you intend to rely on it.
- **Wi-Fi won't connect** — connect to the `Growatt-Bridge-C3` fallback access point and reconfigure credentials via the captive portal.

## License

This configuration is provided as-is. Refer to the licenses of the upstream projects referenced above ([Growatt_ESPHome_ESP32_Modbus_RS485_Example](https://github.com/JasperE84/Growatt_ESPHome_ESP32_Modbus_RS485_Example), [SunspecModbusServer](https://github.com/mahoekst/SunspecModbusServer)) for the components they contributed.
