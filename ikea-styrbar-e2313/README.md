# IKEA STYRBAR E2313 | Main Fixture + Accent Lamp Controller

<p align="center">
  <img src="https://www.zigbee2mqtt.io/images/devices/E2001-E2002-E2313.png" width="230" alt="IKEA STYRBAR E2313">
</p>

A Home Assistant automation blueprint that turns a Zigbee2MQTT IKEA STYRBAR (E2313) into a controller for one room: a main fixture (**Big Light**: power, dimming, colour) and an accent **Lamp** (on/off only). Colour is the default; white is a shortcut.

Forked from the community blueprint [IKEA Styrbar E2313 Two Light Controller with Colour Matching](https://community.home-assistant.io/t/ikea-styrbar-e2313-two-light-controller-with-colour-matching/1018541).

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpqpxo%2Fha-blueprints%2Fblob%2Fmain%2Fikea-styrbar-e2313%2Ftwo_light_controller.yaml)

## What it does

| Gesture | Action |
| --- | --- |
| Top, short press | Toggles the Big Light |
| Bottom, short press | Toggles the Lamp |
| Top, hold | Brightens the Big Light until released (max 30 steps) |
| Bottom, hold | Dims the Big Light until released (max 30 steps) |
| Left arrow, hold | Sweeps the hue smoothly left until released (or the max sweep time) |
| Right arrow, hold | Sweeps the hue smoothly right until released (or the max sweep time) |
| Left arrow, double click | Default white: the bulb's "Color temp startup" value (colour temperature only, brightness untouched) |
| Right arrow, double click | Colour preset 1 |
| Right arrow, triple click | Colour preset 2 (and so on, up to 5 clicks for preset 5) |
| Arrow, single click | Nothing |

Clicks beyond the number of presets are ignored.

## How click counting works

The first arrow click starts a **click window** (default 350 ms). Further clicks of the same arrow inside the window are counted; when the window closes without another click, the action for that count runs. Hold events (`arrow_*_hold`) are separate Z2M actions and are never counted as clicks.

- **Latency:** arrow clicks act up to one click window after the last click. Lower the window for snappier response, raise it if multi-clicking is hard.
- **Automation mode:** `single`. While a click window or hold loop is running, it collects its own follow-up events (more clicks, the release) and other button events are ignored. Top/bottom presses within the click window, or during a hold, are dropped.
- Vertical holds are capped at 30 iterations; the hue sweep is capped by time. Both stop on release.

## Why there are no chords

The Styrbar is a Zigbee remote that reports one action at a time through Zigbee2MQTT (`arrow_left_hold`, `arrow_right_hold`, `brightness_move_up`, and so on). It has no way to report two buttons held together, and holding two buttons simultaneously does not produce a distinct action, so up+down or left+right hold chords cannot be detected. Single clicks, multi-clicks and single holds are used instead.

## Tuning

| Input | Range | Default |
| --- | --- | --- |
| Dimming step | 1-25 % | 10 % |
| Dimming transition (also the delay between loop steps) | 0.1-2.0 s | 0.4 s |
| Hue sweep speed | 10-180 °/s | 60 °/s |
| Max hue sweep time | 5-120 s | 30 s |
| Click window | 200-1000 ms | 350 ms |
| Color temp startup entity | optional number/select/sensor | none |
| Fallback white | 100-600 mireds | 333 (3000 K) |
| Colour presets | list of up to 5 `[hue, saturation]` | red, blue, green, purple, orange |

## Colour handling

Presets and the default white are sent as their own `light.turn_on` command with no transition, then read back and retried up to three times, as in the original blueprint, because many Zigbee bulbs drop colour sent alongside other attributes.

**Default white.** Set the optional *Color temp startup entity* to the bulb's `color_temp_startup` config entity from Zigbee2MQTT. Its state is read as **mireds**, converted to Kelvin (`1000000 / mireds`) for `light.turn_on`, and clamped to the Big Light's own `min/max_color_temp_kelvin`. If the entity is empty or its state is not a number (for example `previous`, `unavailable`), the *Fallback white* input (in **mireds**) is used. The old fixed 2000-6500 K clamp no longer applies; the light's own range is used instead.

**Smooth hue sweep.** Holding left/right runs a loop of `light.turn_on` `hs_color` calls, each fading over one loop interval (the longer of the dimming transition and 0.35 s, so at most about 3 commands per second) so the fades chain. The target hue is computed from the start hue, speed and elapsed time (modulo 360, saturation kept), not read back, because the reported hue lags during a fade. It stops on release or after *Max hue sweep time*. Caveats: some Zigbee bulbs do not fade hue smoothly and will step instead, and command rate limits mean a busy network or slow bulb may stutter.

## Requirements

- Home Assistant 2024.10 or newer
- Zigbee2MQTT with the MQTT integration configured in Home Assistant
- A Big Light that supports brightness, colour and colour temperature, and a Lamp light entity

## Installation

1. Copy `two_light_controller.yaml` into `config/blueprints/automation/pqpxo/`
2. Reload automations, or restart Home Assistant
3. Create an automation from it in **Settings > Automations & Scenes > Blueprints**

## Migration

Breaking change from the two-light version: re-check inputs, re-save each automation and test every button. Removed: two-light colour matching and sync, warm/cool presets, brightness-when-off, transition input, the colour temperature step and arrow temperature shifting, the hue step (replaced by hue sweep speed and max sweep time), the Kelvin default white (replaced by the startup entity plus a mired fallback), and the Konami code and its helpers (unneeded now that colour is the default). The Lamp is on/off only. For two-zone setups stay on the previous version or upstream.

## Other remotes

The action strings are hardcoded to the STYRBAR set (`on`, `off`, `arrow_left_click`, `arrow_right_click`, `arrow_left_hold`, `arrow_right_hold`, `arrow_left_release`, `arrow_right_release`, `brightness_move_up`, `brightness_move_down`, `brightness_stop`). Check what another remote publishes on `zigbee2mqtt/YOUR_DEVICE` and edit them.

## Testing

The YAML parses and the Jinja templates compile, but this has **not** been tested on a live Home Assistant instance or a real STYRBAR.

## Licence

MIT
