# Stop BYD charging from Home Assistant

A Home Assistant package that stops AC charging of a BYD car from HA, without a
smart wallbox. It uses only the on-board charging schedule, written through the
[hass-byd-vehicle](https://github.com/jkaberg/hass-byd-vehicle) integration.

Tested on a **BYD Seal U DM-i** (EU, 11 kW AC, dumb wallbox). Other models may behave differently.

## The trick

The BYD cloud has no "stop charging" command. The car does stop on its own,
usually within 20-60 seconds, when the charging schedule is rewritten like this:

| Field | Value | Why |
|---|---|---|
| start time | **in the future** | a start that already passed keeps the car charging |
| end time | **in the past** (now - 1 min) | this is what ends the session |
| enabled | **true** | a disabled schedule means "charge now" and the end time is ignored |
| repeat | **single** | a daily schedule starts charging again the next day |
| times | full **5-minute** steps | the car rejects or rounds other values |

`script.byd_stop_charging` in [`byd_stop_charging.yaml`](byd_stop_charging.yaml) does exactly that.
If the start time stored in the car has already passed, the script moves it 12 hours ahead.

## What we learned the hard way

Each line cost at least one unwanted charging session.

- **End time in the past with a past start time does not stop the car.** Three attempts in a row failed until the start was moved into the future.
- **A disabled schedule cannot stop anything.** After a single-run window starts, the car disables the schedule by itself, so every stop request must set `enabled: true`.
- **Turning the schedule off starts charging** immediately if the car is plugged in.
- **Any schedule write while plugged in can start charging** for a short time, even when the new window is hours away. Write the planned window while the car is unplugged, and keep the guard running.
- **A window saved while unplugged is not a guarantee.** The car sometimes starts charging the moment it is plugged in. The guard stops it within about a minute.
- **Daily repeat is a trap**: a window planned for Saturday also fired on Friday.
- **The plug state can be stale** after a Home Assistant restart. Poll the car (`fetch_charging`) before writing a schedule.
- **Battery power above zero is not always charging**: regenerative braking looks the same, so the guard requires the plug to be in.
- **Unlocking the car twice also stops charging**, but locking it again resumes it, so it is only useful as a last resort.
- The schedule cannot limit charging to a target percentage. That is done by stopping the session from HA at the target.

## The guard

`automation.byd_charge_guard` reacts to a charging start (charging state, battery
power above 1 kW, plug inserted, plus a check every 2 minutes), calls the stop
script and repeats it every ~3 minutes, up to 10 times, until the car stops.
Add your own condition for "charging is not wanted right now", for example
outside the cheapest hours of a dynamic tariff or after the target charge level is reached.

Results on our car after the last fix: 5 of 5 unwanted starts stopped, four in about 60 s and one in 210 s.
The car takes a few hundred watt-hours at most before it stops.

## Example: planner and dashboard for a dynamic tariff

[`examples/`](examples) holds a cleaned-up version of the planner and dashboard we use with an hourly (dynamic) electricity tariff:

- [`charging_planner.yaml`](examples/charging_planner.yaml): a package with:
  - **Charge modes:** *Auto – cheapest window*, *Charge now*, *Scheduled*, *Off*.
  - **Auto:** finds the cheapest continuous window in the known prices (today and tomorrow) that reaches the target %. Pick a **"charged by" date and time** and the window must end before it; if there is not enough time, the plan says how far it will get.
  - **Charge now / Scheduled:** charge to a separate target, then switch back to Auto.
  - **`sensor.byd_charging_plan`:** the window, kWh, the cost compared with charging right now, and whether it fits the deadline. A window that has started is kept until it ends, so new prices do not cut a session.
  - **`binary_sensor.byd_charging_wanted`:** the condition for the guard, so anything outside the plan gets stopped.
  - **A controller** that starts charging by writing a window covering "now" once charging is wanted and the car is plugged in.
- [`dashboard_charging.yaml`](examples/dashboard_charging.yaml): a Lovelace section with a mode selector, sliders, the deadline picker and a plan summary. Tiles only show in the mode that uses them.

The price sensor needs price lists with `start`, `end` and `value` items (for example Nord Pool from HACS: `raw_today` and `raw_tomorrow`). Charging speed and battery size are variables at the top of the planner (ours: 0.8 %/min below 80 %, 0.6 %/min above, 18.3 kWh).

**Status:** the planner template was checked against our live prices and gives the same window and cost as the version we run. The controller and the dashboard are simplified from ours and have not run in this exact form. Treat them as a starting point.

## Install

1. Install [hass-byd-vehicle](https://github.com/jkaberg/hass-byd-vehicle).
2. Copy `byd_stop_charging.yaml` to `/config/packages/` (packages must be enabled in `configuration.yaml`).
3. Replace `YOUR_BYD_DEVICE_ID`, the `YOUR_byd_*` entity ids and `notify.YOUR_PHONE`.
4. Add your "charging not wanted" condition to the guard, restart HA, turn on `input_boolean.byd_charge_guard`.

## Limits

- Everything goes through the BYD cloud. If the cloud or the car's connection is down, nothing stops the car.
- Detection depends on how often the integration polls. A power meter on the house supply helps to spot charging sooner.
- This is behaviour observed on one car and one firmware. BYD can change it at any time.
- Use at your own risk. This project is not affiliated with BYD.
