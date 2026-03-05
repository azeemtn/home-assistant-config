# Home Assistant — Solar EV Charging Configuration

This repo contains the core Home Assistant configuration for a solar-powered home that automatically manages EV charging based on real-time solar production, battery state, and time of day.

> **Adapt freely to your own setup.** Entity names, amp limits, time windows, and SOC thresholds will all vary by hardware and tariff structure. The logic and patterns here are meant to be a starting point, not a drop-in replacement.

## Overview

**Solar Assistant** monitors the inverter and publishes live power data over MQTT to Home Assistant. HA selectively records that data (to keep the database manageable), computes 2-minute rolling averages for stability, and uses those averages to dynamically control the **Wallbox Pulsar Plus** EV charger — maximising solar self-consumption while protecting the home battery.

The time windows are designed around a **time-of-use electricity tariff**: off-peak overnight rates make it worthwhile to allow heavier grid-powered charging at night, while the daytime windows prioritise solar surplus and battery health.

---

## Hardware (reference setup)

These configs were written for the following hardware. Yours will differ — update entity names accordingly.

| Component | Details |
|---|---|
| Inverter | Sol-Ark (monitored via Solar Assistant) |
| EV Charger | Wallbox Pulsar Plus |
| Solar monitoring | Solar Assistant → MQTT → Home Assistant |

---

## Files

### `configuration.yaml`

Contains sensor definitions, recorder settings, and helper entities used by the automations.

**Recorder** (`purge_keep_days: 30`)

Raw Solar Assistant power sensors (PV, load, grid, battery) update every few seconds and would overwhelm the HA database, so they are **excluded** from recording. Instead, 2-minute rolling averages and energy totals are recorded:

| Recorded entity | Purpose |
|---|---|
| `sensor.solarassistant_battery_state_of_charge` | Battery SOC % |
| `sensor.solarassistant_battery_energy_in/out` | Cumulative energy counters |
| `sensor.*_mean_2m` | Smoothed power values used by automations |
| `sensor.battery_discharge_w_*` | Discharge-direction helpers |

**Statistics sensors** (2-minute rolling mean)

Five `platform: statistics` sensors are defined to smooth out rapid fluctuations before feeding values into automations:

- `sensor.pv_power_mean_2m` — Solar PV output (W)
- `sensor.load_power_mean_2m` — House load (W)
- `sensor.grid_power_mean_2m` — Grid power (W)
- `sensor.battery_power_mean_2m` — Battery power (W, + = charging, − = discharging)
- `sensor.wallbox_charging_power_mean_2m` — EV charger draw (kW)

**Template sensors** (battery discharge direction helpers)

Solar Assistant's battery power sign convention can vary by inverter setup. Two template sensors are provided — use whichever one reads correctly for your hardware:

- `Battery discharge W, discharge positive` — use if discharge shows as positive watts
- `Battery discharge W, discharge negative` — use if discharge shows as negative watts

**Input booleans** (manual override switches)

| Entity | Purpose |
|---|---|
| `input_boolean.ev_max_charge_override` | Force EV to charge at 40A immediately |
| `input_boolean.ev_throttle_active` | Throttle EV to 8A immediately |

---

### `automations.yaml`

Contains two automations.

---

#### 1. Wallbox Auto Reload (`wallbox_auto_reload`)

The Wallbox cloud integration occasionally drops out, causing sensors to go `unavailable`. This automation detects that and automatically reloads the integration without requiring manual intervention.

- **Trigger:** Any Wallbox sensor unavailable for 2+ minutes
- **Action:** Reloads the Wallbox config entry, waits 30 seconds, logs the result

---

#### 2. EV Solar Controller (`ev_solar_controller`)

The main automation. Runs every 3 minutes and sets the Wallbox charging current (6–40A in 4A steps) based on available solar surplus, battery SOC, and time of day.

**Surplus calculation**

```
true_surplus = pv_power - (load_power - ev_power)
```

The EV's own draw is subtracted from load before calculating surplus, so the controller correctly sees what solar power is actually *available for* the EV rather than treating the EV's consumption as a competing load.

**Charging limits:** min 6A · max 40A · step 4A · assumed voltage 240V

**Decision priority (evaluated in order)**

| Priority | Condition | Action |
|---|---|---|
| 1 | `ev_max_charge_override` ON | Force **40A** |
| 2 | `ev_throttle_active` ON | Force **8A** |
| 3 | Battery SOC ≥ 98% | Force **40A** (battery full, dump surplus to EV) |
| 4 | Time window logic | See table below |
| — | Car not plugged in / not charging | Skip, do nothing |

**Time windows**

| Window | Hours | Behaviour |
|---|---|---|
| Night | 00:00–07:00 | Fixed **20A** — cheap off-peak grid rate, top up the car overnight |
| School | 07:00–09:00 | Fixed **40A** — quick top-up before school departure |
| Morning | 09:00–12:00 | Solar-follow, reserve 1000W to battery first, minimum 6A |
| Priority | 12:00–16:00 | Tiered battery reserve by SOC (see below), battery never discharges to power EV |
| Evening | 16:00–21:00 | SOC-aware solar-follow, battery never discharges to power EV |
| Pre-night | 21:00–24:00 | Fixed **10A** — winding down before off-peak window opens at midnight |

**Priority window (12:00–16:00) — tiered battery reserve**

Peak solar hours. Battery is always filled first; EV only gets what's left over.

| Battery SOC | Reserve for battery | EV behaviour |
|---|---|---|
| < 60% | 4000W | Step down unless battery getting ≥ 4000W |
| 60–80% | 3000W | Step down unless battery getting ≥ 3000W |
| 80–98% | 2000W | Step down unless battery getting ≥ 2000W |
| ≥ 98% | — | Caught by battery-full override → 40A |

Amps step up by 4A when the battery is meeting its reserve target and there is surplus remaining. Amps step down by 4A when the battery is discharging or not meeting its target.

**Evening window (16:00–21:00) — SOC-aware solar-follow**

Solar is fading; protect battery for evening house loads.

| Battery SOC | Behaviour |
|---|---|
| < 80% | Fixed **6A** regardless of solar |
| 80–90% | Solar-follow with 1000W battery reserve; step down if battery discharges |
| ≥ 90% | Solar-follow with no reserve; step down if battery discharges |

**Logging**

Every run logs a full status line to the HA logbook:
```
hour, pv, load, ev, surplus, soc, batt_pwr, batt_reserve, batt_full,
batt_meeting_target, car_charging, current_amp, override_max, override_throttle, window
```

---

## Key entity reference

| Entity | Description |
|---|---|
| `sensor.solarassistant_pv_power` | Raw PV power from Solar Assistant (W) |
| `sensor.solarassistant_load_power` | Raw house load from Solar Assistant (W) |
| `sensor.solarassistant_grid_power` | Raw grid power from Solar Assistant (W) |
| `sensor.solarassistant_battery_power` | Raw battery power from Solar Assistant (W) |
| `sensor.solarassistant_battery_state_of_charge` | Battery SOC (%) |
| `sensor.pv_power_mean_2m` | 2-min averaged PV power (W) |
| `sensor.load_power_mean_2m` | 2-min averaged load (W) |
| `sensor.battery_power_mean_2m` | 2-min averaged battery power (W) |
| `sensor.wallbox_charging_power_mean_2m` | 2-min averaged EV draw (kW) |
| `number.wallbox_pulsar_plus_sn_1331793_maximum_charging_current` | Wallbox amp setpoint |
| `switch.wallbox_pulsar_plus_sn_1331793_pause_resume` | Wallbox pause/resume (on = charging) |
| `sensor.wallbox_pulsar_plus_sn_1331793_status_description` | Wallbox status text |
