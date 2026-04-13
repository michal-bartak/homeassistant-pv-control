Recently I noticed several PV automation projects seen the light of day. All of them consider managing of energy accumulation devices (ie hot water tanks). My project focuses solely on battery charge and discharge, based on weather forecast and incoming spot prices. It's a response to my needs after changing flat rate selling price to spot.

While it's implemented with specific dependencies in mind, the article provide general thoughts allowing to adjust for use with any other PV inverter.

> :sparkles: Some parts of code has been proposed by AI. Also text has been corrected with help of AI.

# Introduction

The idea of a household PV plant as a money-making machine was never really accurate, even if advertisements sometimes make it seem that way. Anyone who had that expectation was likely brought back to reality fairly quickly.

In fact, household's PV systems are often only marginally profitable - unless they are also viewed as a hobby. The initial investment is quite high. Larger systems perform better outside the summer season, but they are also more expensive (especially due to batteries). In most cases, the return on investment is not expected earlier than 10 years.

ROI depends on several factors. Let’s break them down.

# Understanding PV Plant ROI

The most important fact is that the gross price of purchased electricity is several times higher than the price at which electricity is sold. In the Czech Republic, households sell electricity at net prices. While no regulatory costs are applied on the selling side, intermediary companies charge various handling fees, reducing the net income.

At the same time, purchasing electricity includes additional regulatory costs such as distribution, network maintenance, and special taxes - all of which are further increased by VAT.

Here is the evolution of total selling and purchasing prices over the past years:

![Sale vs Purchase Price Comparison](images/sale_vs_purchase.png)

The ROI (Return on Investment) is built on two main components:

- **PV energy consumed by the household** – This should form the largest portion of ROI. It represents the cost of electricity you did not have to buy thanks to your PV system. The more PV energy you can store and use within the household, the better.
- **PV energy sold** – Typically, especially during summer months, more PV produced energy is sold than utilized, but its total contribution to ROI is significantly smaller. Doesn't mean negligible.

Given this, income from sold energy is only a fraction of total ROI. In my household, the ratio was approximately 1:4 in 2025.

![ROI build-up](images/roi_buildup.png)

There are several ways to manage electricity prices. In most countries, it is possible to choose between flat rates and spot pricing for both buying and selling independently.

Using spot prices for both (so-called “spot surfing”) is likely the most efficient approach. However, it requires adapting the entire household to shift consumption to the cheapest time periods. I guess not every family is ready for that. We are not.

I opted for a hybrid approach: buying electricity at a flat rate (preferably fixed for several years, especially in today’s uncertain environment) and selling at spot prices.

Selling at spot prices opens opportunities for some optimizations. This article presents a simple yet effective method of using spot prices and weather forecasts to extract additional value from a PV system.

To make this work, reliable data sources are essential. Spot prices are officially published in advance. They have to be imported to Home Assistant. For PV production forecasting, **Solcast** is the best solution I know. It offers a free tier for hobby users (with some limitations) and works very accurately for my location.

<img src="images/prices_and_sun.png" alt="Prices and Irradiance" width="70%">

The upper graph shows spot prices. Next-day prices are typically published in the afternoon, around 2 PM.

Daily price patterns usually feature two peaks: morning and evening, and a trough around midday. The peaks are typically short, while the midday dip can sometimes last for several hours.

The lower graph shows predicted PV production based on data from Solcast. The forecast is updated multiple times per day, improving accuracy over time. Solcast provides prediction for several days ahead giving overview what is comming next, though these are not utilized by this project.

<img src="images/solcast_7d.png" alt="Solcast 7 days forecast" width="60%">

# The Automation Concept

These observations lead to a set of automation rules:

1. Sell energy stored in batteries during peak price periods
2. Delay battery charging during morning hours, to sell for good prices
3. Charge the battery during the cheapest price window
4. Do not discharge if not enough PV energy is forecasted
5. Secure minimum energy in battery that is needed for normal household operations at any time

It starts from creation of 3 template sensors, that calculates time windows for charging and discharging. The code incolved for their creation does the most dirty job. the result is utilized by the automation as well as visualization.

They need a few settings:
<img src="images/config_1.png" alt="Config 1" width="40%">

Sensors are updated every minute, then once start horizon is reached, the particuluar sensor is locked until next midnight. Sensors are also protected against moving the target to the past.

## Discharge

To sell stored energy, the inverter must be switched to a mode that exports battery energy to the grid. This may be available as a main operating mode or as a scheduled mode (Time of Use, ToU).

Discharging must not result in an energy deficit later. Therefore no discharge is triggered if no enough PV energy is forecasted as well as minimum battery levels must be maintained. In my setu it makes 70% SOC after evening discharge and 25% SOC reserved after morning discharge.

> Selling energy despite low forecast only makes sense if selling prices exceed buying prices. This does happen occasionally, but not often enough to rely on.

## Delayed Charge

It is the period between morning discharge and the cheapest charging window.

During this time:
- The system should behave similarly to General mode, but
- PV production should prioritize export to the grid over charging the battery.

This is typically called **Feed-in Mode**. If unavailable, limiting the charging current may help achieving a similar result, though Feed-in Mode still allows charging if PV production exceeds export limits.

> Wattsonic Gen3 with firmware 2.x does not provide Feed-in Mode. It was introduced in later firmware versions.

This period implements several guards:

- it's is activated only if proceeds the morning discharge. Otherwise not enough solar energy forecasted is considered

- in this mode the battery still provides energy to the hausehold, in case PV doesn't cover the requirement. If battery SOC drops to the predefined limit, the mode is interrupted and inverter returns to regular operational mode.

- if spot prices drop too low during this period, selling does not make sense anymore. It is better to use PV energy charge the battery.

## Cheapest Charge

Charging during the cheapest hours requires no special inverter mode - just the standard “General” mode. More important is securing charge window big enough to fully charge battery. During development and testing cycle I introduced some parameters to save from extreme sitatuations:

- Chargin End Time - If cheapest hours become later than usual and/or due to bad weather the chargin requires more time, both cases together with forecast unpredictability might lead to situation that battery will never fully charge. This setting effectively shifts the time window back, preventing too late charging.

- Charge Overhead - increases energy demand for charging by given factor. It extrapolates household energy usage during the same hours.

<img src="images/config_2.png" alt="Config 2" width="40%">

# The Package

The entire solution is implemented as a Home Assistant package, and it depends on some external components.

```mermaid
flowchart
subgraph container [" "]
    subgraph ext [" "]
        direction LR
            sc[Solcast]
            sp[Spot Prices]
            inv[Inverter]
            inv ~~~ sc
            inv ~~~ sp
    end
    subgraph ha [PV Control Package]
        direction LR
        ts[Template Sensors] --> a[Automation]
        inv --> a
        cnf[Config Entities] --> a
        a <--> as["Automation State"]
        a -->|call| scr[Scripts]
        scr --> inv
        sc --> a
        sp --> a
    end
end
style container fill:white,stroke-width:0px
style ext fill:white,stroke-width:0px
```

The package creates the following entities, all of them are prefixed with `pv_ctrl`.

**Script**
The `script.pv_ctrl_inverter` is a single script, parametrized to implement inverter-specific commands. It's called by the automation. It might be used for manual control of the automation or inverter from dashboard.

Example:
```
action: script.pv_ctrl_inverter
data:
  mode: general
```

Following modes are accepted:
**general** - Resets inverter to general mode, incl. restoring unlimited battery charge
**discharge_grid** - enables discharge mode to the grid. Currently implemented with use of schedule Wattsonic feature, that allows to chose discharge mode. 
**feedin** - make inverter prefer injecting produced energy to the grid, instead of charging battery. Achieved by limiting charge current. 
**charge_disabled** - sets charge current to zero (unused)
**charge_enabled** - sets charge current to maximum (unused)
**injection_enabled** - Enables injection to the grid
**injection_disabled** - Disables injection to the grid

Two latest operations are utilized by another - independent - automation that prevents selling energy when its price is below the limit.

**Template Sensors**
These sensors calculate start and end of time periods, reflecting automation life-cycle.

- `sensor.pv_ctrl_most_expensive_hours_morning` – Start time of morning discharge
- `sensor.pv_ctrl_most_expensive_hours_afternoon` – Start time of evening discharge
- `sensor.pv_ctrl_cheapest_hours` – Start time of cheapest charging window

All sensors above consist of additional data under attributes.data key. The information is used for the internal logic as well as for dashboard.

**Automation**
The `automation.pv_ctrl_executor` automation is the core component. It implements finite automata, making decissinos based on inputs provided by template sensors mentioned above, as well as state of inverter.

All of them must be adapted if you are using a different inverter than Wattsonic/Solinteg.

**Configuration Entities**

| Entity                                       | Description |
|----------------------------------------------|-------------|
| `input_boolean.pv_ctrl_edit_mode`            | Used for dashboard only, preventing accidental changes to the settings. |
| `input_select.pv_ctrl_mode`                  | Allows to enable the automation either in `real` or `dry mode`, or `disable` it. The `Dry-run` does everything but requesting changes to the inverter. It's good to test if the automation phases proceed as expected. |
| `input_boolean.pv_ctrl_debug`                | Toggles recording the debug informations to the Home Assistant log |
| `input_select.pv_ctrl_phase`                 | Materializes the current automation phase. Not intended to be edited manually. Possible phases are `General`, `Morning Discharge`, `Delayed Charge`, `Cheapest Charge`, `Evening Discharge`. |
| `input_number.pv_ctrl_min_suncast_current_day` | Minimum forecasted energy for today required to allow the morning discharge |
| `input_number.pv_ctrl_min_suncast_next_day` | Minimum forecasted energy for tomorrow required to allow the evening discharge |
| `input_number.pv_ctrl_soc_limit_morning`    | SOC limit for morning discharge (25%) |
| `input_number.pv_ctrl_soc_limit_evening`    | SOC limit for evening discharge (70%) |
| `input_number.pv_ctrl_min_export_price`     | Maximum energy price (per MWh), that prevents exporting energy (e.g., 250 CZK). |
| `input_number.pv_ctrl_charge_velocity`      | Expected charging power, the velocity the battery can be charged with. |
| `input_number.pv_ctrl_discharge_velocity`   | Expected discharge power, used to calculate a time needed to discharge battery to requested SOC |
| `input_number.pv_ctrl_battery_capacity`     | Used in calculation of 1% of SOC |
| `input_datetime.pv_ctrl_charge_delay_time_limit` | Limits predicted end time of cheapest charge time window. Might be helpful if cheapest hours (occasionally) starts late afternoon, but you don't want to delay charging so much |


---
- pv_ctrl_charge_forecast_overhead



## External Entities

The automation make use of following entities, that come from other integrations:

### Solcast

- `sensor.solcast_pv_forecast_forecast_today`   - sensor that provides forecasted PV energy today
- `sensor.solcast_pv_forecast_forecast_tomorrow` - sensor that provides forecasted PV energy tomorrow

### Spot Prices

- `sensor.current_spot_electricity_price_15min` - sensor providing current spot price

### Inverter

- `sensor.wattsonic_battery_soc` - Inverter sensor that provides the state of charge of the battery

# Dashboard

![The Dashboard](images/dashboard.png)

The design of the dashboard is a matter of personal preference. This one helps visualize relationships between variables and allows manual control in case the automation behaves unexpectedly (which can happen during development).

Configuration controls are protected by an Edit Mode toggle to prevent accidental changes - especially useful especially on mobile devices.

The top buttons control inverter modes. The five buttons below represent automation phases.

This dashboard uses:
- `custom:apex-charts` for graphs
- `custom:button-card` for controls
- `custom:restriction-card` for locking UI elements

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

The Home Assistant package is available under Github [link](https://github.com/michal-bartak/homeassistant-pv-control/blob/main/packages/pv_control.yaml)

**Requirements**
* Wattsonic gen3 integration ([github](https://github.com/GiZMoSK1221/hass-addons))
* Solcast ([github](https://github.com/BJReplay/ha-solcast-solar))
* CZ Spot prices ([github](https://github.com/rnovacek/homeassistant_cz_energy_spot_prices))
