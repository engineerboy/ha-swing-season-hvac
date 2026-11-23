# Swing-Season HVAC for Home Assistant

Automatic heat ↔ AC switching for climates where you can need **both on the same day**, sometimes more than once. Built for comfort first, with guards so the heat and AC never fight.

Most thermostats' "auto" (heat/cool) modes assume real seasons: they want a wide gap between the heat and cool setpoints, and in practice they let the house drift several degrees before switching. In a climate where you can run the AC on New Year's Day and the heat on a spring morning, that means someone in the house is constantly feathering the thermostat by hand. This package does that job.

<!-- Screenshots: add sanitized images to docs/ and reference them here, e.g. ![Control tab](docs/control.png) -->

## How it decides

The key idea: **the setpoints are comfort targets, not switching thresholds.** The thermostat is only ever in heat *or* cool (never the thermostat's own heat/cool mode), so only one setpoint is active at a time and the heat and cool targets can sit 1°F apart, or even equal, without fighting. Switching has its own rules and evidence.

The rules are deliberately **lopsided toward AC** (my household strongly prefers too-cool over too-warm; flip the defaults if yours is the opposite):

| Decision | Rule (defaults) |
|---|---|
| Stay on AC | The default. AC sitting idle in a slightly cool house costs nothing. |
| Switch to heat | House ≥ 2°F below the heat target **for 45 min**, **and** the projected outdoor high over the next 6 h is ≤ 72°F. |
| Emergency heat | House ≥ 4°F below the heat target for 15 min, regardless of forecast. |
| Back to AC | House ≥ 1°F above the cool target for 10 min while the heat is *not* running (ignores heating overshoot), **or** the house is at its heat target and a warm-up is forecast. |
| Equipment protection | At least 20 min between switches (emergency heat excepted). |

**Projected outdoor high** = your own outdoor sensor's current reading + how much the hourly forecast says it will warm over the lookahead window. The forecast supplies the *trend*, your sensor supplies the *level*, which matters when your yard runs several degrees off the official station.

**Outdoor temperature** uses your own sensor, but if it disagrees with a second source (e.g. a nearby weather station) by more than 5°F, the median of the three sources (yours, the station, the forecast's current temperature) wins, so one bad sensor can't take over.

## Features

- **Schedule per mode:** three morning steps (warm up for the day) and three evening steps (cool down for sleep), separately for heat and AC. Targets are *computed* from the schedule, so a restart or a missed time self-corrects.
- **One writer:** a single automation is the only thing that changes the thermostat. It runs every 5 minutes and at startup, and re-applies the schedule if anything drifted.
- **Verify and retry:** every command is checked against the physical thermostat and re-sent once if it didn't take. Mode changes wait for the thermostat to confirm the new mode before the temperature is sent (some thermostats reject a cool setpoint while still in heat).
- **Cloud-failure recovery:** with Versatile Thermostat, if a command was silently dropped (e.g. a cloud API error), the automation notices the physical thermostat disagrees and re-sends it directly.
- **Guards:** a thermostat turned off (by a person or a restart) goes straight back to the right mode; a preset selected by anyone is cleared and the schedule restored.
- **Notifications** that explain *why*: every heat/AC switch with the reason and the temperatures behind it, plus an optional "verbose" switch for every setpoint change.
- **Dashboard** with Control, Schedule, Tuning, and History tabs, including a plain-English "what it's doing and why" line.

## Requirements

- Home Assistant 2024.10 or newer (uses the `triggers:` / `actions:` syntax).
- A thermostat in Home Assistant that supports separate `heat` and `cool` modes. I run it on [Versatile Thermostat](https://github.com/jmcollin78/versatile_thermostat) layered over a Nest, so a future thermostat swap is a one-line change in VT. A plain `climate` entity works too; the VT-specific recovery step simply never runs.
- At least one indoor temperature sensor (two or more is better; they're averaged).
- An outdoor temperature sensor, and ideally a second outdoor source.
- One or two weather integrations that provide an **hourly** forecast (e.g. NWS, Open-Meteo).
- A notification service.

## Install

1. **Enable packages** if you haven't already. In `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `packages/swing_season_hvac.yaml` to `/config/packages/`.
3. **Point it at your entities.** Either rename your entities in Home Assistant to the IDs below (easiest, and makes future thermostat swaps trivial), or find-and-replace them in the file:

   | Placeholder | Your entity |
   |---|---|
   | `climate.house_thermostat` | your thermostat (Versatile Thermostat or plain) |
   | `sensor.indoor_temperature_1`, `_2` | indoor temperature sensors (add more to the list in the indoor sensor) |
   | `sensor.outdoor_temperature_local` | your own outdoor sensor |
   | `sensor.outdoor_temperature_station` | a second outdoor source |
   | `weather.forecast_primary` | weather entity with an hourly forecast |
   | `weather.forecast_backup` | a second one, used if the first fails |
   | `notify.notify` | your notification service |

   The indoor sensor list is `[entity, max age in hours]`. Sensors that only report when the value changes (many Zigbee sensors) need a longer max age, or they'll be dropped as stale on a stable day.
4. Check the configuration (**Developer tools → YAML → Check configuration**), then restart.
5. Run **`script.swing_season_hvac_load_defaults` once.** It loads the tuning defaults and a sample schedule. (The settings deliberately have no `initial:` value, which would reset your tuning on every restart.)
6. Add the dashboard: create a new dashboard, open the **Raw configuration editor**, and paste `dashboards/comfort_dashboard.yaml` (with the same entity substitutions).
7. Set your schedule on the **Schedule** tab.
8. Turn off any schedules, Eco/away modes or "auto" features in the thermostat's own app, so it isn't making decisions too.
9. Watch the **Control** tab for a while: `sensor.climate_v2_desired_mode` shows what it *would* do. When you're happy, turn on **Automatic control** (`input_boolean.climate_v2_in_control`).

**Taking over manually:** turn Automatic control off. While it's on, manual setpoint changes are reset to the schedule within 5 minutes.

If you use Versatile Thermostat, give its presets sensible temperatures even though this package doesn't use them; anyone can still select one from the thermostat dialog or a voice assistant.

## Tuning

Everything on the **Tuning** tab can be changed live. The wait times (45/15/10/15 min) are the `delay_on` values on the decision sensors in the package file.

Things worth knowing:

- **Forecast lookahead** is how far ahead "projected high" looks. A short lookahead (e.g. 3 h) reacts to the next few hours; a long one (12 h) lets a warm afternoon block morning heat.
- The projected high adds the forecast's warm-up to your *current* local reading. On mornings when your sensor already reads well above the official station (fog, valleys, urban heat), part of the forecast's rise is the station catching up, so the projection can run a few degrees high. That's a conservative error for an AC-first household; tune the heat limit accordingly.
- A heat pump's backup heat strips aren't visible to Home Assistant through most thermostat integrations ("heating" is all you get).

## How it's built

| Piece | Entity |
|---|---|
| Indoor average | `sensor.climate_v2_indoor_temperature` |
| Outdoor (with sanity check) | `sensor.climate_v2_outdoor_temperature` |
| Scheduled targets | `sensor.climate_v2_cool_target`, `sensor.climate_v2_heat_target` |
| Projected high | `sensor.climate_v2_outdoor_high_projected` |
| Decision signals | `binary_sensor.climate_v2_heat_allowed_by_forecast`, `_heat_needed`, `_emergency_heat`, `_too_warm_in_heat`, `_warm_day_ahead` |
| The decision | `sensor.climate_v2_desired_mode` |
| Forecast updater | `automation.climate_v2_update_forecast_outlook` (every 30 min) |
| The only writer | `automation.climate_v2_apply_hvac_mode_and_setpoint` |

## Lessons learned the hard way

- **Cloud thermostats drop commands.** A Google Nest API error once left the thermostat 2°F off schedule all night while Home Assistant showed the right value. Hence verify-and-retry, and comparing against the physical thermostat rather than any layer above it.
- **Order matters on a mode switch.** Sending the mode and temperature together made the Nest reject the temperature (still in heat), and it cooled to the old heat setpoint. Now the temperature waits for the mode to confirm.
- **Restarts briefly report "off".** Without the immediate restore, that looked like the system turning itself off.
- **Don't drive switching off the setpoints.** An early version used "below the cool target" as "might need cooling", which in winter would have toggled heat and AC every couple of hours.

## Ideas / not yet done

- Room sensors in each main room, with fan-only circulation when the spread between rooms gets too large.
- Self-regulation so the *room average* reaches the target instead of the thermostat's own sensor.
- Supply/return air temperature probes for equipment health.

## Disclaimer

Provided as-is, with no warranty. This controls real HVAC equipment: watch it closely for the first days, and keep the minimum time between switches sensible for your equipment.

## License

MIT, see [LICENSE](LICENSE).
