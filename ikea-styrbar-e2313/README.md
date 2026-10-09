# IKEA STYRBAR E2313 | Main Fixture + Accent Lamp Controller

<p align="center">
  <img src="https://www.zigbee2mqtt.io/images/devices/E2001-E2002-E2313.png" width="230" alt="IKEA STYRBAR E2313">
</p>

A Home Assistant automation blueprint that turns a Zigbee2MQTT IKEA STYRBAR (E2313) into a controller for one room: a main fixture (**Big Light**: power, dimming, colour temperature and optional colour) and an accent **Lamp** (on/off only).

Forked from the community blueprint [IKEA Styrbar E2313 Two Light Controller with Colour Matching](https://community.home-assistant.io/t/ikea-styrbar-e2313-two-light-controller-with-colour-matching/1018541). The two-zone layout, matched turn-on and colour sync of the original are gone; see [Migration](#migration-from-the-two-light-version).

## What it does

| Gesture | White mode | Colour mode |
| --- | --- | --- |
| Top, short press | Toggles the Big Light | same |
| Bottom, short press | Toggles the Lamp | same |
| Top, hold | Brightens the Big Light until released (max 30 steps) | same |
| Bottom, hold | Dims the Big Light until released (max 30 steps) | same |
| Left arrow | Colour temperature warmer (clamped 2000-6500K) | Hue down by the hue step |
| Right arrow | Colour temperature cooler (clamped 2000-6500K) | Hue up by the hue step |
| Konami code | Switch to colour mode | Switch to white mode |

## Colour mode

Colour mode behaves like the original blueprint in how colour is sent: a hue change is its own `light.turn_on` call with `hs_color` and no transition, then read back and retried up to three times, because many Zigbee bulbs drop colour sent alongside other attributes.

Where the mode is stored:

- **With a colour mode helper** (an `input_boolean`): the helper is the source of truth, so the mode survives the light being turned off.
- **Without a helper**: the mode is inferred from the Big Light itself, which is in colour mode when its `color_mode` is a colour mode (hs, xy, rgb, rgbw, rgbww). Switching only takes effect while the light is on.

If the Big Light does not support any colour mode, colour mode is ignored and the arrows always shift colour temperature.

When the mode is switched with the Konami code the light, if on, is moved immediately: to its current hue (default orange) when entering colour mode, or to its current colour temperature (default 4000K) when going back to white.

## Konami code

Press, as short presses: **top, top, bottom, bottom, left, right, left, right**.

Detection needs an `input_text` helper (max length at least 30). Each press of the sequence stores `<matched count>|<timestamp>` in it; a press that continues the sequence within the **Konami code timeout** (time between presses, default 3 s) advances it, anything else restarts it (a press of the first button counts as a new start). On the eighth press the mode is toggled, progress is reset and that final press does nothing else. Without the helper, the Konami code is disabled and everything else works.

Because normal presses must keep working, the first seven presses perform their usual action while entering the code. The pairs cancel out (Big Light toggled twice, Lamp toggled twice, temperature or hue stepped back and forth), so you may see a brief flicker, and a clamped colour temperature at 2000K or 6500K can end slightly off.

## Tuning

| Input | Range | Default |
| --- | --- | --- |
| Dimming step | 1-25 % | 10 % |
| Dimming transition | 0.1-2.0 s | 0.4 s |
| Colour temperature step | 100-1200 K | 400 K |
| Hue step | 5-90 ° | 30 ° |
| Konami code timeout | 1-10 s | 3 s |

## Requirements

- Home Assistant 2024.10 or newer (the blueprint uses input sections)
- Zigbee2MQTT with the MQTT integration configured in Home Assistant
- A Big Light that supports brightness and colour temperature (and colour for colour mode)
- A Lamp light entity
- Optional helpers: an `input_boolean` for colour mode and an `input_text` for the Konami code

## Installation

Click the import badge, or add it manually:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpqpxo%2Fha-blueprints%2Fblob%2Fmain%2Fikea-styrbar-e2313%2Ftwo_light_controller.yaml)

1. Copy `two_light_controller.yaml` into `config/blueprints/automation/pqpxo/`
2. Reload automations, or restart Home Assistant
3. Create an automation from it in **Settings > Automations & Scenes > Blueprints**

## Implementation notes

- **Capability checks are computed inline.** `supported_color_modes` is a list of `StrEnum` members; storing the list itself in a template variable breaks membership tests on some integrations. The blueprint only stores the resulting boolean.
- **`mode: restart`** lets a release or any new button event cancel a running hold loop. A very fast sequence of presses can therefore interrupt the previous press's action.

## Other remotes

The action strings are hardcoded to the STYRBAR set: `on`, `off`, `arrow_left_click`, `arrow_right_click`, `brightness_move_up`, `brightness_move_down`. For another Z2M remote, check what it publishes on `zigbee2mqtt/YOUR_DEVICE` and edit the strings in the `choose` conditions and the `code` variable.

## Migration from the two-light version

This is a breaking change. Re-check the inputs, re-save every automation and test each button. The warm/cool colour presets, colour matching and colour sync are removed, and the Lamp is on/off only. For two-zone setups stay on the previous version or the upstream blueprint.

## Troubleshooting

**Nothing happens on a button press.** Confirm the controller name matches the Z2M friendly name exactly and that the actions arriving match those above.

**The Konami code does nothing.** Check that the helper is an `input_text` with a long enough max length, and look at the automation trace: `new_idx` should count up with each press within the timeout.

**Hue steps do not stick.** The trace will show three retries. Something else (Adaptive Lighting, another automation) is undoing the colour.

## Licence

MIT
