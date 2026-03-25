# Introduction

The idea of a household PV plant as a money-making machine was never really accurate, even if advertisements sometimes make it seem that way. Anyone who had that expectation was likely brought back to reality fairly quickly.

In fact, PV systems are often only marginally profitable—unless they are also viewed as a hobby. The initial investment is quite high. Larger systems perform better outside the summer season, but they are also more expensive (especially due to batteries). In most cases, the return on investment is not expected earlier than 10 years.

ROI depends on several factors. Let’s break them down.

# Understanding PV Plant ROI

The most important fact is that the gross price of purchased electricity is several times higher than the price at which electricity is sold. In the Czech Republic, households sell electricity at net prices. While no regulatory costs are applied on the selling side, intermediary companies charge various handling fees, reducing the net income.

At the same time, purchasing electricity includes additional regulatory costs such as distribution, network maintenance, and special taxes—all of which are further increased by VAT.

Here is the evolution of total selling and purchasing prices over the past years:

![Sale vs Purchase Price Comparison](images/sale_vs_purchase.png)

The ROI (Return on Investment) is built on two main components:

- **PV energy consumed by the household** – This should form the largest portion of ROI. It represents the cost of electricity you did not have to buy thanks to your PV system. The more PV energy you can store and use within the household, the better.
- **PV energy sold** – Typically, more energy (in kWh) is sold, but its total contribution to ROI is significantly smaller.

Given this, income from sold energy is only a fraction of total ROI. In my household, the ratio was approximately 1:4 in 2025.

![ROI build-up](images/roi_buildup.png)

There are several ways to manage electricity. In most countries, it is possible to choose between flat rates and spot pricing for both buying and selling independently.

Using spot prices for both (so-called “spot surfing”) is likely the most efficient approach. However, it requires adapting the entire household to shift consumption to the cheapest time periods—not every family is ready for that. We are not.

I opted for a hybrid approach: buying electricity at a flat rate (preferably fixed for several years, especially in today’s uncertain environment) and selling at spot prices.

Selling at spot prices opens opportunities for optimization. This article presents a simple yet effective method of using spot prices and weather forecasts to extract additional value from a PV system.

To make this work, reliable data sources are essential. Spot prices are officially published and are often already integrated into Home Assistant. For PV production forecasting, Solcast is the best solution I know. It offers a free tier for hobby users (with some limitations) and works very accurately for my location.

<img src="images/prices_and_sun.png" alt="Prices and Irradiance" width="70%">

The upper graph shows spot prices in the Czech Republic. Next-day prices are typically published in the afternoon, around 2 PM.

Daily price patterns usually feature two peaks—morning and evening—and a trough around midday. The peaks are typically short, while the midday dip can sometimes last for several hours.

The lower graph shows predicted PV production based on data from Solcast. The forecast is updated multiple times per day, improving accuracy over time.

# The Automation Concept

These observations lead to a set of automation rules:

1. Sell energy stored in batteries during peak price periods
2. Delay battery charging and sell PV production during morning hours
3. Charge the battery during the cheapest price window

This sounds simple, but requires proper setup. First, your PV inverter must be integrated with Home Assistant in a way that allows control over its operating modes. Available features may depend on the inverter model and firmware version.

## Discharge

To sell stored energy, the inverter must be switched to a mode that exports battery energy to the grid. This may be available as a main operating mode or as a scheduled mode (Time of Use, ToU).

Discharging must not result in an energy deficit later. As discussed, buying electricity is much more expensive than selling it. Therefore, minimum battery charge levels must be maintained.

In my setup:
- 70% SOC is reserved after evening discharge
- 25% SOC is reserved after morning discharge

This works well for my household needs.

> It is possible to sell more energy in the evening at the expense of morning discharge. This may be more profitable since evening prices are usually higher. However, splitting discharge into two phases provides a buffer for unexpected evening or night consumption (e.g., washing, gaming).

Energy shortages can also result from poor weather. If low PV production is expected the next day, it is better to keep energy stored rather than sell it.

I use Solcast forecast data for this purpose. It provides daily energy predictions as well as hourly breakdowns. While fine-grained optimization is possible, daily totals are sufficient in my experience.

Discharge is triggered only if the forecasted energy exceeds a defined threshold. But how should this threshold be calculated?

In principle, it needs to cover three things:

1. **Daily household consumption** – the PV production must be sufficient to power the household throughout the day
2. **Battery recharge** – there must be enough excess energy to recharge the battery back to 100%
3. **Safety margin** – an additional buffer to account for forecast inaccuracies and to avoid purchasing energy from the grid

The margin is important because PV forecasts are never perfect, and household consumption may vary. Without it, the system may end up buying expensive electricity later.

In practice, this threshold does not need to be calculated with high precision. A conservative estimate based on typical daily consumption and battery size is usually sufficient. In my case, a fixed value of **20 kWh** works reliably.

> Selling energy despite low forecast only makes sense if selling prices exceed buying prices. This does happen occasionally, but not often enough to rely on.

## Charge

Charging during the cheapest hours requires no special mode—just the standard “General” mode.

## Delayed Charge

The most complex part is the period between morning discharge and the cheapest charging window.

During this time:
- The system should behave similarly to General mode
- PV production should prioritize export to the grid over charging the battery.

This is typically called **Feed-in Mode**. If unavailable, limiting the charging current may achieve a similar effect, though with slight differences. For example, Feed-in Mode still allows charging if PV production exceeds export limits.

> Wattsonic Gen3 with firmware 2.x does not support Feed-in Mode. It was introduced in later firmware versions.

If spot prices drop too low during this period, selling may no longer be profitable. In that case, it is better to start charging the battery earlier to prepare for potential household consumption.

## Optimal Time Windows

Finally, we need to determine optimal time windows for charging and discharging.

These depend on:
- The amount of energy to transfer
- The rate of charging/discharging

In practice, these rates vary due to household consumption and PV output. We may assume that the variation is not significant enough to require precise modeling.

Therefore, calculations are done approximately, using 15-minute intervals and fixed transfer rate. The result is the start time of each window, while the end is determined by reaching target SOC levels.

# The Package

The entire solution is implemented as a Home Assistant package. All entities are prefixed with `pv_ctrl`.

The `pv_ctrl_executor` automation is the core component. It calls the `pv_ctrl` script, which implements inverter-specific commands.

Both must be adapted if you are using a different inverter than Wattsonic/Solinteg.

The package creates the following entities:

## Configuration Entities

- `input_boolean.pv_ctrl_active` – Enables or disables the automation
- `input_boolean.pv_ctrl_debug` – Enables or disables recording debug informations to the Home Assistant log
- `input_select.pv_ctrl_phase` – Tracks the current automation phase. Not intended to be edited manually.
- `input_number.pv_ctrl_min_suncast_tomorrow` – Minimum forecasted energy required to allow discharge
- `input_number.pv_ctrl_soc_limit_morning` – SOC limit for morning discharge (25%)
- `input_number.pv_ctrl_soc_limit_evening` – SOC limit for evening discharge (70%)
- `input_number.pv_ctrl_min_export_price` – Maximum energy price (per MWh), that prevents exporting energy (e.g., 250 CZK).

## Template Sensors

- `sensor.pv_ctrl_most_expensive_h_morning` – Start time of morning discharge
- `sensor.pv_ctrl_most_expensive_h_afternoon` – Start time of evening discharge
- `sensor.pv_ctrl_cheapest_h_start` – Start time of cheapest charging window

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

The dashboard design is a matter of personal preference. This one helps visualize relationships between variables and allows manual control in case the automation behaves unexpectedly (which can happen during development).

Configuration controls are protected by an Edit Mode toggle to prevent accidental changes—especially useful especially on mobile devices.

The top buttons control inverter modes. The five buttons below represent automation phases.

This dashboard uses:
- `custom:apex-charts` for graphs
- `custom:button-card` for controls
- `custom:restriction-card` for locking UI elements

# The Rabbit Hole

During development, it’s very easy to fall into the “just one more feature” spiral. The model presented here is intentionally kept as simple as possible, but there are many directions where it could be extended. For example:

- defining different energy requirements for each day of the week  
- using more granular Solcast data (e.g. hourly) to better estimate battery charging time  
- calculating (dis)chargin time windows with higher precission
- pick up most expensive time periods instead of continuous blocks
- operating fully in the energy/power domain instead of relying on SOC and estimated transfer rates  
- planning additional loads for the next day
- introducing a “vacation mode” to reflect lower household consumption  
- integrating with a calendar to anticipate changes in usage
- incorporating AI to predict household consumption based on historical data

So, is it worth it?

From a learning and development perspective—definitely yes. It’s a great playground for experimenting and improving your setup.

From a financial perspective, it’s less clear.

More advanced logic means more complex code, more configuration, and more edge cases to handle—such as vacations, holidays, or unusual household behavior. Higher precision and tighter margins also require more accurate predictions, especially for household consumption, which is inherently difficult to model. In practice, this can even place additional expectations on other household members.

And in the end, the improvement in ROI may be marginal.

That said, it always depends on the specific setup. In some cases, the extra complexity might pay off—but it’s worth considering whether the added effort is justified.