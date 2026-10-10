# Amber + FoxESS solar dashboard for Home Assistant

An iPad-friendly Home Assistant dashboard for a home with **Amber Electric** (dynamic pricing) and a **FoxESS** solar/battery inverter. It shows live solar, home, grid and battery numbers next to Amber price and Solcast forecast charts. It also includes an automation that finds the cheapest time to charge an EV, and simple heat/cool automations for an air conditioner.

Everything fits on one screen with no scrolling. It was built for a 13" iPad in landscape and shrinks automatically on smaller screens.

> Unofficial, shared as-is. Not affiliated with Amber, FoxESS, IKEA, Hisense or Home Assistant. Check values (especially cost estimates) against your own bills.

## What's in here

| File | What it is |
|---|---|
| `dashboards/ipad_layout.yaml` | The main one-screen layout (one card; use it in a **Panel** view) |
| `dashboards/power_ledger_dashboard.yaml` | A multi-tab YAML dashboard (history view + Solar Overview) |
| `automations/calculate_cheapest_ev_charging_window.yaml` | Every 30 min, finds the cheapest charging window from Amber's forecast |
| `automations/ac_auto_heat_cool.yaml` | Four automations (heat on/off, cool on/off) with hysteresis |
| `themes/power_ledger_dark.yaml` | Dark amber/teal theme |

## Requirements

**Integrations:** Amber Electric (built into Home Assistant), a FoxESS Cloud integration that gives you `daily_generation`, `pv_power`, `soc` etc. sensors, Solcast PV Forecast, and a weather entity.

**HACS frontend cards:** `mushroom`, `mini-graph-card`, `apexcharts-card`, `card-mod`, `weather-chart-card`, `digital-clock`.

Optional: a Hisense/ConnectLife climate entity and temperature sensors (the AC automations), any EV (the charging card only displays a time; it doesn't control the car).

## Setup

### 1. Replace the placeholders

Search each file for `YOUR_` and replace it with your own entity names. Find them under **Tools → States** (older versions: Developer Tools → States).

| Placeholder | What to put there |
|---|---|
| `YOUR_AMBER_SITE` | The prefix of your Amber sensors, e.g. `sensor.<site>_general_price` |
| `YOUR_FOX_SERIAL` | The prefix of your FoxESS sensors, e.g. `sensor.<serial>_pv_power` |
| `YOUR_AC` | Your air-con climate entity |
| `YOUR_WEATHER_ENTITY` | Your weather entity |
| `YOUR_HALLWAY_TEMPERATURE`, `YOUR_BEDROOM_TEMPERATURE`, `YOUR_LIVINGROOM_TEMPERATURE` | Room temperature sensors |

Also check these values that are hard-coded for my house:
- **Battery size** is 20 kWh (`soc / 100 * 20` in the Battery card).
- **Standing charge** is `0.00` in the Break-even cards. Put your daily supply charge there.
- **Panel names** (Garage / House) are labels for PV1 / PV2 / PV3.
- **AC thresholds** are in each automation (heat on below 18°, off above 19°; cool on above 25°, off below 18°).

### 2. Create the helpers

- **EV charging window:** `input_datetime.ev_start` and `input_datetime.ev_end` (both with **date and time** enabled), and `input_number.ev_charge` (hours of charging you need).
- **Daily home energy:** a *Riemann sum integral* helper on your load-power sensor named `FoxESS Total Load Energy`, then a *Utility Meter* (daily) named `FoxESS Daily Load Energy`. Leave the **metric prefix empty** if your power sensor is already in kW, otherwise you get a `kkWh` unit that Home Assistant won't accept.
- **AC control temperature:** a *Template sensor* named `AC Control Temperature` (unit °C, device class temperature) with this state template. It uses the room sensor, and falls back to the AC's own reading if the room sensor drops out:

  ```
  {% set t = states('sensor.YOUR_HALLWAY_TEMPERATURE') %}{{ t | float if is_number(t) else state_attr('climate.YOUR_AC', 'current_temperature') }}
  ```

### 3. Install the files

- **Themes:** put `power_ledger_dark.yaml` in a `themes/` folder in your config and make sure `configuration.yaml` has `frontend: themes: !include_dir_merge_named themes`.
- **Layout:** add a dashboard view, set **View type** to **Panel (1 card)**, add a **Manual** card and paste the contents of `dashboards/ipad_layout.yaml`.
- **Multi-tab dashboard (optional):** save it in your config folder and register it:

  ```yaml
  lovelace:
    dashboards:
      power-ledger:
        mode: yaml
        filename: dashboards/power_ledger_dashboard.yaml
        title: Power Ledger
        icon: mdi:transmission-tower
  ```
- **Automations:** add each one as its own automation (**Edit in YAML**, one block at a time, without the `---` lines).

## Things I learned the hard way

- **Amber forecast times are UTC.** If you write them into an `input_datetime` as-is, the time is hours off. The EV automation converts with `as_local`.
- **Triggers on a changing value don't fire for a value that is already true when you save.** The AC automations check every 5 minutes with a time pattern instead.
- **Battery IKEA sensors can drop offline.** That's why the AC automations read the fallback helper above.
- **After changing dashboards, reset the app's frontend cache** if you don't see the change.
- **Break-even is an estimate:** today's kWh × the *current* price, not a true running total.

## Smaller screens

`ipad_layout.yaml` scales itself down on shorter screens using card-mod's CSS `zoom`. The numbers are at the top of the file; lower them if a screen still scrolls.
