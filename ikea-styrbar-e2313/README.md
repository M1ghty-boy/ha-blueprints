# IKEA STYRBAR E2313 | Main Light + Lamp Controller

<p align="center">
  <img src="https://www.zigbee2mqtt.io/images/devices/E2001-E2002-E2313.png" width="230" alt="IKEA STYRBAR E2313">
</p>

Customised two-light control for the IKEA STYRBAR, for one dimmable, multicolour light (such as the "big light" on a main fixture) and one fixed-brightness light (such as a lamp on a smart plug).

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpqpxo%2Fha-blueprints%2Fblob%2Fmain%2Fikea-styrbar-e2313%2Ftwo_light_controller.yaml)

## ⚠️ AI Slop Disclosure

> "This fork, and most of this readme, is 100% vibecoded AI slop. I had a new bedroom with a light and a main light switch that wasn't functioning. I usually endeavour for my creations to be human written or human architected and supervised, but i do not understand YAML, nor do i have the time to fiddle with it for weeks when i need lighting. This was 100% (actually pretty decently) led by github copilot under my loose instruction for feature iteration. This slip in my standards is purely as i made this for myself. I would not subject the general public to slop created in this manner. As such, if you use this, it is all at your own risk"

## What changed from the original, and why

This is a fork of the community blueprint [IKEA STYRBAR E2313 | Two Light Controller with Colour Matching](https://community.home-assistant.io/t/ikea-styrbar-e2313-two-light-controller-with-colour-matching/1018541) (source: [pqpxo/ha-blueprints](https://github.com/pqpxo/ha-blueprints)).

The original is a two-zone, colour-matching controller: the left and right arrows toggled two separate lights, the second light inherited the first one's brightness and colour, short vertical presses dimmed, and vertical holds set hard-coded warm and cool presets. It also had an optional live colour sync.

This version is for a single room with one multicolour main light and one lamp, so the layout was rebuilt around that:

- **Single-room layout.** The top button toggles the Big Light and the bottom button toggles the Lamp (on/off only). Vertical holds brighten and dim the Big Light.
- **Colour-first arrows.** Colour is the default; white is a shortcut. Holding an arrow sweeps the hue, a left double click sets the default white, and right-arrow multi-clicks select colour presets. Single clicks do nothing.
- **Colour presets.** A customisable list of up to five `[hue, saturation]` colours (defaults: red, blue, green, purple, orange).
- **Smooth hue sweep.** A time-based loop of `hs_color` calls whose fades chain together, with a speed slider and a maximum sweep time.
- **Startup white.** The default white is read from the bulb's own "Color temp startup" setting (in mireds), with a fallback value for when that is unavailable.
- **Tuning sliders.** Dimming step, dimming transition, hue sweep speed, maximum sweep time and click window.
- **Removed features.** Two-light colour matching and colour sync, warm/cool presets, the brightness-when-off input, the transition input, arrow colour-temperature shifting and its step, and the experimental Konami code and its helpers. They only make sense for two zones, or are unnecessary now that colour is the default.

## What it does

| Gesture | Action |
| --- | --- |
| Top, short press | Toggles the Big Light |
| Bottom, short press | Toggles the Lamp |
| Top, hold | Brightens the Big Light until released (maximum 30 steps) |
| Bottom, hold | Dims the Big Light until released (maximum 30 steps) |
| Left arrow, hold | Sweeps the hue smoothly left until released (or the maximum sweep time) |
| Right arrow, hold | Sweeps the hue smoothly right until released (or the maximum sweep time) |
| Left arrow, double click | Default white: the bulb's "Color temp startup" value (colour temperature only, brightness untouched) |
| Right arrow, double click | Colour preset 1 |
| Right arrow, triple click | Colour preset 2 (and so on, up to six clicks for preset 5) |
| Arrow, single click | Nothing |

Clicks beyond the number of presets are ignored.

## How click counting works

The first arrow click starts a **click window** (default 350 ms). Further clicks of the same arrow inside the window are counted; when the window closes without another click, the action for that count runs. Hold events (`arrow_*_hold`) are separate Zigbee2MQTT actions and are never counted as clicks.

- **Latency:** arrow clicks act up to one click window after the last click. Lower the window for a snappier response, or raise it if multi-clicking is difficult.
- **Automation mode:** `single`. While a click window or hold loop is running, it collects its own follow-up events (more clicks, the release) and other button events are ignored. Top and bottom presses made during a click window or a hold are dropped.
- Vertical holds are capped at 30 iterations and the hue sweep is capped by time. Both stop on release.

## Why there are no chords

The STYRBAR is a Zigbee remote that reports one action at a time through Zigbee2MQTT (`arrow_left_hold`, `arrow_right_hold`, `brightness_move_up`, and so on). It cannot report two buttons held together, so up+down or left+right hold chords cannot be detected. Single clicks, multi-clicks and single holds are used instead.

## Tuning

| Input | Range | Default |
| --- | --- | --- |
| Dimming step | 1-25 % | 10 % |
| Dimming transition (also the delay between loop steps) | 0.1-2.0 s | 0.4 s |
| Hue sweep speed | 10-180 °/s | 60 °/s |
| Max hue sweep time | 5-120 s | 30 s |
| Click window | 200-1000 ms | 350 ms |
| Colour temp startup entity | optional number, select or sensor | none |
| Fallback white | 100-600 mireds | 333 (3000 K) |
| Colour presets | list of up to five `[hue, saturation]` pairs | red, blue, green, purple, orange |

## Colour handling

Presets and the default white are sent as their own `light.turn_on` command with no transition, then read back and retried up to three times, as in the original blueprint, because many Zigbee bulbs drop colour sent alongside other attributes.

**Default white.** Set the optional *Colour temp startup entity* to the bulb's `color_temp_startup` config entity from Zigbee2MQTT. Its state is read as **mireds**, converted to Kelvin (`1000000 / mireds`) for `light.turn_on`, and clamped to the Big Light's own `min_color_temp_kelvin` and `max_color_temp_kelvin`. If the entity is empty or its state is not a number (for example `previous` or `unavailable`), the *Fallback white* input (in **mireds**) is used. There is no fixed 2000-6500 K clamp; the light's own range is used.

**Smooth hue sweep.** Holding left or right runs a loop of `light.turn_on` `hs_color` calls, each fading over one loop interval (the longer of the dimming transition and 0.35 s, so at most about three commands per second) so that the fades chain. The target hue is calculated from the start hue, speed and elapsed time (modulo 360, saturation kept) rather than read back, because the reported hue lags during a fade. It stops on release or after *Max hue sweep time*. Caveats: some Zigbee bulbs do not fade hue smoothly and will step instead, and command rate limits mean a busy network or slow bulb may stutter.

## Requirements

- Home Assistant 2024.10 or newer
- Zigbee2MQTT with the MQTT integration configured in Home Assistant
- A Big Light that supports brightness, colour and colour temperature, and a Lamp light entity

## Installation

1. Copy `two_light_controller.yaml` into `config/blueprints/automation/pqpxo/`.
2. Reload automations, or restart Home Assistant.
3. Create an automation from it in **Settings > Automations & Scenes > Blueprints**.

## Migration

This is a breaking change from the two-light version: re-check the inputs, re-save each automation and test every button. The removed features are listed [above](#what-changed-from-the-original-and-why). The Lamp is on/off only. This is not suitable for two-zone setups; for those, stay on the previous version or the upstream blueprint.

## Other remotes

The action strings are hard-coded to the STYRBAR set (`on`, `off`, `arrow_left_click`, `arrow_right_click`, `arrow_left_hold`, `arrow_right_hold`, `arrow_left_release`, `arrow_right_release`, `brightness_move_up`, `brightness_move_down`, `brightness_stop`). Check what another remote publishes on `zigbee2mqtt/YOUR_DEVICE` and edit them.

## Testing

The YAML parses and the Jinja templates compile, this is in active use in my oqn live Home Assistant instance with a real STYRBAR. The riskiest assumptions are that the startup entity's state is in mireds, that the bulb fades hue smoothly, and that `wait_for_trigger` matches the release events.

## Licence

The author does not want anyone earning money from this AI-generated slop, so commercial use is not allowed. It'd be a waste of energy, and hypocritical for me not to leave this for everyone's free use otherwise. The new and changed work in this repository is licensed under the [PolyForm Noncommercial License 1.0.0](../LICENSE): free to copy, modify and share for noncommercial purposes.

Scope, stated plainly: the upstream blueprint by pqpxo has no licence file, and its README states only "MIT". The portions of this fork that derive from it remain under that author's own terms, which I cannot change; the noncommercial licence applies only to the new and changed work. Nothing here claims more than that, and this is not legal advice.
