# Introduction 
> :sparkles: Some bits of code has been proposed by AI. Also text has been corrected with help of AI.

After A year or so of having photovoltaic installation (PV) and pretty good flat rate of selling energy I had to switch to spot prices. It introduced a need to squeeze as much as possible from this setup. in terms of optimizing a return of investment (ROI).

The solution does not implement automation of activating household appliances, that accumulates energy during cheapest periods. It correlates electricity spot prices with solar exposure forecast to find the best time frames to sell the energy as well as charge battery.

# Understanding PV Plant ROI

> :bulb: the information below is based on Central European market. Might differ in other parts of the world.

The most important factor is that the gross price of purchased electricity is several times higher than the price at which electricity is sold. In the Czech Republic, households sell electricity at net prices. While no regulatory costs are applied on the selling side, intermediary companies charge various handling fees, reducing the net income.

At the same time, purchasing electricity includes additional regulatory costs such as distribution, network maintenance, and special taxes - all of which are further increased by VAT.

Here is the evolution of total selling and purchasing prices over the past years:

![Sale vs Purchase Price Comparison](images/sale_vs_purchase.png)
You might spot, that since Nov'25 the saling price is not constant anymore. Correct, those are spot prices I opted since then. It's obvious that average price is higher than previously flat rate. Red color indicates prices below which export to grid should avoided. It's because of negative prices spot prices shifted by a fee for the company who I sell the energy to.

The ROI (Return on Investment) is built on two main components:

- **PV energy consumed by the household** – This should form the largest portion of ROI. It represents the cost of electricity you did not have to buy thanks to your PV system. The more PV energy you can store and use within the household, the better.
- **PV energy sold** – Typically, especially during summer months, more PV produced energy is sold than utilized, but its total contribution to ROI is significantly smaller. Doesn't mean negligible.

# The Automation Concept

These observations lead to a set of general automation rules:

1. Sell energy stored in batteries during peak price periods
2. Delay battery charging during morning hours, to sell PV production for good prices
3. Charge the battery during the cheapest price window
4. Discharge only if enough PV energy is forecasted
5. Secure minimum energy in battery to avoid buying it for household operations

This together introduces four phases:

## Discharge (morning and evening)

> ??? To sell stored energy, the inverter must be switched to a mode that exports battery energy to the grid. This may be available as a main operating mode or as a scheduled mode (Time of Use, ToU).

While it could be limited to single discharge, split into two has benefits:
* prices during evening hours are ussually a bit higher
* Due to narrowness of peaks it's better to share energy over two peaks
* split into two gives better margin management
* morning PV forecast allows last minute decission to skip the morning discharge

Discharging must not result in an energy deficit later. Therefore no discharge is triggered if not enough PV energy is forecasted. At the same time a minimum battery levels must be maintained. In my setup it makes 25% SOC reserved after morning discharge together with minimum of 17kWh forecasted energy or 70% SOC after evening discharge and 13 kWh of forecasted PV energy next day.

The setting of edge conditions might differ from installation to installation. It must be set with safe margin to avoid buying energy, for insuficient sunlight weather.

## Delayed Charge

It is the period between morning discharge and the cheapest charging window.

During this time:
- The system should behave similarly to General mode, but
- PV production should prioritize export to the grid over charging the battery.

This period implements several guards:

- it's activated only if proceeds the morning discharge. It comes from assumption that, if discharge has been skipped, then it's gonna be no enough PV energy to play with.

- during this period, the battery still provides energy to the hausehold, in case PV doesn't cover the needs. If battery SOC drops to the predefined limit, the mode is interrupted and inverter returns to regular operational mode, securing battery charge from PV.

- if spot prices drop undel limit, selling does not make sense anymore. Instead of suppressing PV production, it is better to use it to charge the battery.

> ??? This mode might be called **Feed-in Mode**. If unavailable, limiting the charging current may help achieving a similar result, though Feed-in Mode still allows charging if PV production exceeds export limits.

> Wattsonic Gen3 with firmware 2.x does not provide Feed-in Mode. It was introduced in later firmware versions.

## Cheapest Charge

Charging during the cheapest hours requires no special inverter mode - just the standard “General” mode. More important is securing enough time to fully charge battery. During development and testing cycle I introduced some parameters to avoid extreme sitatuations:

- Charging End Time - If cheapest hours become later than usual and/or bad sun conditions requires more time to charge, both cases together with forecast unpredictability might lead to situation that battery will not charge enough. This setting prevents it by planning the chargin time window earlier.

- Charge Overhead - Normally the demand is calculated from battery size, SOC to charge and forecasted PV energy capped by Charge velocity. It does not take into account household usage. This parameter increases energy demand for charging by given factor in an attempt to predict realistic time window for the charge.

<img src="images/config_2.png" alt="Config 2" width="40%">

# The Package

For no deployment better option, the entire solution is implemented as a Home Assistant package:

![alt text](images/diagram.svg)

**Proxy Sensors**
Consider them as an API between 3rd party integration sensors, respresenting state of inverter, solar forecast and prices. Thanks to this approach, the code never references 3rd party sensors, being independent from their data format. Also, in case of replacing integrations, it's enough to adjust proxy sensors.

**Script**
Like proxy sensors, the `script.pv_ctrl_inverter` plays a role of the proxy for calling the solar inverter.
It's a single, but paramterized script, that implements inverter-specific commands.

Example:
```yaml
action: script.pv_ctrl_inverter
data:
  mode: general
```

Following modes are accepted:
`general` - Resets inverter to general mode, incl. restoring unlimited battery charge
`discharge_grid` - enables discharge mode to the grid. Currently implemented with use of schedule Wattsonic feature, that allows to chose discharge mode. 
`feedin` - make inverter prefer injecting produced energy to the grid, instead of charging battery. Cyrrently chieved by limiting a charge current. 
`charge_disabled` - sets charge current to zero (unused)
`charge_enabled` - sets charge current to maximum (unused)
`injection_enabled` - Enables injection to the grid
`injection_disabled` - Disables injection to the grid

Two latest operations are utilized by another - independent - automation that prevents selling energy when its price is below configured limit.

**TimeWindow Sensors**
These template sensors calculate start and end of charging and discharging time periods for the current day

- `sensor.pv_ctrl_most_expensive_hours_morning` – time of morning discharge
- `sensor.pv_ctrl_most_expensive_hours_afternoon` – time of evening discharge
- `sensor.pv_ctrl_cheapest_hours` – time of cheapest charging window

These sensors store additional data under attributes.data key. The results are used by automation as well as are presented on the dashboard.

**Automation State and Config Entities**

Both are implemented as `input` entities. Automation state materializes state of the automation logic.Config Entities provide option to customize and control the automation.


| Entity                                       | Description |
|----------------------------------------------|-------------|
| `input_boolean.pv_ctrl_edit_mode`            | Used for dashboard only, preventing accidental changes to the settings. It's especially important for mobile views, where current HA UI makes an accidental change of parameters more then likely |
| `input_select.pv_ctrl_mode`                  | Allows to enable the automation either in `real` or `dry mode`, or `disable` it. The `Dry-run` does everything but requesting changes to the inverter. It's good to test if the automation phases proceed as expected. |
| `input_boolean.pv_ctrl_debug`                | Toggles recording the debug informations to the Home Assistant log |
| `input_select.pv_ctrl_phase`                 | Materializes and provides the current automation phase. Not intended to be edited manually. Possible values are `General`, `Morning Discharge`, `Delayed Charge`, `Cheapest Charge`, `Evening Discharge`. |
| `input_number.pv_ctrl_min_suncast_current_day` | Minimum forecasted energy for today; required for the morning discharge |
| `input_number.pv_ctrl_min_suncast_next_day` | Minimum forecasted energy for tomorrow; required for the evening discharge |
| `input_number.pv_ctrl_soc_limit_morning`    | SOC limit for morning discharge |
| `input_number.pv_ctrl_soc_limit_evening`    | SOC limit for evening discharge |
| `input_number.pv_ctrl_min_export_price`     | Maximum energy price (per kWh), that prevents exporting energy (e.g., 0.25 CZK). |
| `input_number.pv_ctrl_charge_velocity`      | Maximum charging power, the velocity the battery can be charged with. Used to cap PV energy provided by Solcast |
| `input_number.pv_ctrl_discharge_velocity`   | Maximum discharge power, used to calculate a time needed to discharge battery to requested SOC |
| `input_number.pv_ctrl_battery_capacity`     | Used in calculation of 1% of SOC |
| `input_datetime.pv_ctrl_charge_delay_time_limit` | Limits predicted end time of cheapest charge time window. Might be helpful if cheapest hours (occasionally) starts late afternoon, but you don't want to delay charging so much |

**The Automation**
Finally, the `automation.pv_ctrl_executor` is the core component that makes decissions based on its current state and inputs provided by sensors.

## External Entities

The solution make use of following entities, that come from other integrations:

**Solcast**

- `sensor.solcast_pv_forecast_forecast_today`   - sensor that provides forecasted PV energy today
- `sensor.solcast_pv_forecast_forecast_tomorrow` - sensor that provides forecasted PV energy tomorrow

**Spot Prices**

- `sensor.current_spot_electricity_price_15min` - sensor providing spot prices.

**Inverter**

- `sensor.wattsonic_battery_soc` - Inverter sensor that provides the state of charge of the battery

# Dashboard

![The Dashboard](images/dashboard.png)

The design of the dashboard is a matter of personal preference. This one grown during development to help visualization of relationships between variables and to allow manual control in case the automation behaves unexpectedly (which can happen during development).

Configuration controls are protected by an Edit Mode toggle to prevent accidental changes - especially useful on mobile devices.

The top buttons control inverter modes. The five buttons below represent automation phases.

This dashboard uses:
- `custom:apex-charts` for graphs
- `custom:button-card` for controls
- `custom:restriction-card` for locking UI elements
- `cardmod` integration

# The Rabbit Hole

During development, it’s very easy to fall into the “just one more feature” spiral. The model presented here is intentionally kept as simple as possible, but there are many directions where it could be extended. For example:

- defining different energy requirements for each day of the week  
- pick up most expensive time periods instead of continuous blocks
- operating fully in the energy/power domain instead of relying on SOC and estimated transfer rates
- planning additional loads for the next day
- introducing a “vacation mode” to reflect lower household consumption
- integrating with a calendar as a provider of anticipated changes in usage
- incorporating AI to predict household consumption based on historical data

So, is it worth it?

From a learning and development perspective, maybe the answer is yes. It’s a great playground for experimenting and improving your setup.

From a financial perspective, it’s less clear.

More advanced logic means more complex code, more configuration, and more edge cases to handle. Higher precision and tighter margins also require more accurate predictions, especially for household consumption, which is inherently difficult to model. In practice, this can even place additional expectations on other household members.

And in the end, the improvement in ROI may be marginal.

That said, it always depends on the specific setup. In some cases, the extra complexity might pay off - but it’s worth considering whether the added effort is justified.

# The code

The Home Assistant package is available under Github [link](https://github.com/michal-bartak/homeassistant-pv-control/blob/main/packages/pv_control.yaml).
In case you are not familiar with HA packages, here is the [documentation](https://www.home-assistant.io/docs/configuration/packages/).

**Requirements**
* Wattsonic gen3 integration by GiZMoSK ([GitHub](https://github.com/GiZMoSK1221/hass-addons))
* Solcast ([GitHub](https://github.com/BJReplay/ha-solcast-solar), [HA forum](https://community.home-assistant.io/t/wattsonic-photovoltaic-power-plant-fve-integration/406135))
* CZ Spot prices ([GitHub](https://github.com/rnovacek/homeassistant_cz_energy_spot_prices))

> :exclamation: It's important to maintain normalized unit magnitude for all sensors and input values: kW / kWh

Reconfiguring for other coverters is possible. It requires to adjust the script and proxy sensors, maintaining the same input/output behavior.
