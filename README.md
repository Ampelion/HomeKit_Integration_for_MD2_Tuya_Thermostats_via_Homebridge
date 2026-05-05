# Herschel iQ MD2 Thermostat → HomeKit (via Homebridge / Tuya) with °F Override
Herschel iQ MD2 Thermostat (SmartLife/Tuya) integration to HomeKit

A working Homebridge configuration that exposes the **Herschel iQ MD2 Wired Thermostat** to Apple HomeKit with correctly-handled Fahrenheit temperature reporting.

The Herschel iQ MD2 works with the SmartLife app for control; Smart Life is the consumer-facing rebrand of the Tuya IoT platform, so the device is reachable through 
any Tuya-compatible bridge — including Homebridge with the `homebridge-tuya-platform` plugin. This repo documents the deviceOverride schema needed to make HomeKit 
show correct °F values.

SmartLife app is tedious to set up and not a fit for a dynamic household that does things at different times of day.

## The problem this solves

Tuya cloud devices report state via numeric "Data Points" (DP IDs) without semantic labels — DP 1, DP 2, DP 3, etc. The Homebridge Tuya plugin reads these and, by 
default, interprets thermostat temperature values as °C. The Herschel iQ MD2, when configured for Fahrenheit display, reports temperature as a plain Fahrenheit integer
 — so without an override, a `76°F` reading from the device gets interpreted as `76°C`, which HomeKit then converts and displays as `169°F`. Wildly wrong, and nothing 
in the device's own UI hints at what's gone sideways.

This repo documents the override schema that tells the plugin to treat the integer as Fahrenheit directly. The DP ID translation is from the on-device settings.  they are fiddly to get 
so here they are:

## DP ID translation for Herschel iQ MD2

| DP ID | Type | Mapping | Notes |
|-------|------|---------|-------|
| 1 | Boolean | `switch` (power on/off) | Used |
| 2 | Integer | `temp_set_f` (target temp in °F) | Used. Range 41–113°F. Reported as plain °F integer. |
| 3 | Integer | `temp_current_f` (current temp in °F) | Used. Range 32–122°F. Reported as plain °F integer. |
| 4 | String | Mode (e.g., "Manual") | Ignored |
| 7 | Boolean | Possibly child lock or oscillation | Ignored |
| 13 | Integer | Unused | — |
| 14 | String | State string (e.g., "no_working") | Ignored |
| 19 | String | Temperature unit ("f" = Fahrenheit) | Confirms unit |
| 20 | Integer | Unknown — possibly humidity or alternate temp | Unmapped |
| 21 | Integer | Possibly correctly formatted °F | Unmapped |

If your MD2 is configured for Celsius display the picture may differ — the DP layout itself is fixed by the firmware, but the `scale` override in the schema would need 
to change accordingly. This table reflects an MD2 set to °F.

## What the override schema does

The relevant fragment from `config.example.json`:

```json
{
    "code": "temp_set_f",
    "type": "Integer",
    "dpid": 2,
    "unit": "℉",
    "min": 41,
    "max": 113,
    "scale": 0,
    "step": 1,
    "onSet": "temp_set_f",
    "onGet": "temp_set_f"
}
```
onSet and onGet were the most important setting for the temperautures to render properly in HomKit.

Key fields:

- `dpid: 2` — the Tuya data point this entry maps
- `code: "temp_set_f"` — the semantic label HomeKit will use
- `unit: "℉"` — **the critical bit**: declares the unit so HomeKit doesn't apply a Celsius interpretation to a Fahrenheit integer
- `scale: 0` — keeps the integer as the actual reading with no division
- `min` / `max` — clamp range, in °F
- `onSet` / `onGet` — bind setter and getter to the same code (so writes from HomeKit and reads from the device share the mapping)

## Setup

1. **Install Homebridge** if you haven't already: https://homebridge.io
2. **Install the Tuya plugin**: `npm install -g @0x5e/homebridge-tuya-platform` (or via the Homebridge UI)
3. **Set up a Tuya IoT developer account** at https://iot.tuya.com:
   - Create a Cloud project (Project Type 2 / Smart Home)
   - Get your Access ID and Access Key from the project's overview page
   - Link your Smart Life account to the project under **Devices → Link App Account**
4. **Find your device IDs**:
   - In the Tuya IoT console: your project → Devices
   - Copy the device ID for each MD2 you want to expose
5. **Copy `config.example.json` to `config.json`** in your Homebridge config directory
6. **Fill in your credentials and device IDs** in `config.json`:
   - `accessId`, `accessKey` — from your Tuya project
   - `username`, `password` — your Smart Life account
   - `deviceOverrides[].id` — your MD2 device IDs from step 4
7. **Restart Homebridge.** The thermostats should appear in HomeKit with correct °F readings.

## Adding more thermostats

Each MD2 needing the override gets its own entry in the `deviceOverrides` array. The schema is identical across MD2 units — copy the `schema` block and just change the `id`. My setup runs four MD2s on the same schema.

## Notes

- The `category: "wk"` value classifies these as thermostat-like devices in Tuya's taxonomy. The Tuya plugin uses category to pick a default HomeKit accessory type; `wk` maps to thermostat.
- `debug: true` at the platform level enables verbose logging — useful while you're working out DP mappings, can be turned off once everything is stable.
- If you don't already have a Tuya IoT developer project set up, the [Homebridge Tuya plugin's README](https://github.com/0x5e/homebridge-tuya-platform) has a more detailed walkthrough of the developer-portal setup (including the "Link App Account" step that's easy to miss).

## Acknowledgments

Built on top of `homebridge-tuya-platform` by 0x5e and contributors. The `deviceOverrides` mechanism that makes this all possible is documented (sparsely) in that plugin's repo.
