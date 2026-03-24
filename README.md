# Introduction

The PV Plant as money maker machine was never the case, even thought it might look like from advestisements. Who had that dream, the life must have validated it quickly.
In fact, it's on the edge of profitability, unless considered as part of a hobby. 
PV initial investment is pretty high. The bigger PV covers seasons other than the summer better. But are more expensive (especially batteries). Most often investment is expected to be returned not earlier than after 10 years. 

Return of Investment depends on several factors. Let break them down quickly.

# Understanding FV Plant ROI

Most important fact is: that gross price of purchased energy is several times higher than sold energy. In Czech Republic, households sell energy for net price. While no other regulatory costs are involved, intermediary companies requiest some handling fees in different form, reducing net income. AT the same time, buying energy involves additional regulatory costs, ie distribution, network maintanance, special tax. And all of them are extended by VAT. 

Here is evolution of total sale and purchase prices for recent year or so:

![Sale vs Purchase Price Comparison](images/sale_vs_purchase.png)

The ROI (Return of Investment) is build up based on two factors:
- PV energy consumed by a household - It should be the biggest part of ROI. It contributes to ROI with cost of energy you DID NOT buy thanks to PV. The more PV energy is accumulated in some form in the house, the better. 
- PV energy sold - it's expected that that more energy in energy units is sold, but in total it will contribute to ROI way less.

Knowing the above, it's clear that income from sold energy is a fraction of total ROI. For my household consumption, it made about 1:4 ratio for 2025.

![ROI build-up](images/roi_buildup.png)

There are several ways how operate with electric energy. I assume, in most countries it's possible to choose between flat rate and spot prices, for selling and purchasing independently.
Using spot prices for both operation (spot surfing) is probably the most effective approach, but requires to adapt the whole household, to shift consumption to moments of chiepest prices. I'm not sure every familly is ready for that. We are not. 

I opt in an intermediate option: buy energy for flat rate (better if fixed for several years, especially in today's hard times) and sell on spot.
Selling on spot opens an opportunity for optimizations attemps. And here we go, this article covers a simple yet effective approach of using spot prices and weather predictions to squeeze a few bucks more out of PV.

To make it happen, we need reliable data source. Spot prices are published officially. Possibly everyone is integrated into HA today. For FV plant predition, Solcast is the best I know. It offers free tier for amateurs like us (with slight limitations). It works hell preciselly for my location.

<img src="images/prices_and_sun.png" alt="Proces and Irradiance" width="70%">

The upper graph presents spot prices (in Czech Republic). Next day prices are provided afternoon, at about 2pm. 
Daily prices evolution characterizes ussually with two peeks: morning and evening and one deep around the mid day. Peaks are ussuaaly narrow, while the deep may sometimes cover half a day.

The bottom graph renders prediction of energy to be obtained from FV plant. It comes from Solcast service, and is updated several times a day improving its prediction.

# The concept

The above becomes a cue for automation rules:
1. to sell energy accumulated in batteries during peak hours
1. to delay battery charge, selling PV production during morning hours
1. to charge battery durring cheepest price period

Seems pretty easy. Before start, you need to have your PV Inverter integrated with HA in the way, that allows control of its operation modes. Accessibility of some features might depend on the inverter model or even its firmware version.

**Discharge**

To sell accumulated energy, the inverter has to be switched into mode that flushes the battery to the grid. It might be available as main inverter mode or as scheduled mode (ToU - Time of Use). 

Obviously, discharging should not lead to energy deficit during following time period. As discussed earlier, purchase price is way higher than selling one. We need to ensure the battery-stored energy can cover household operations. For this reason, we use certain levels of battery charge, the discharge process cannot exceed.  It's the matter of configuration. In my case, leaving 70% after evening discharge and 25% after morning one nails the needs.

> It's possible to sell more energy on the evening, sacrificing morning discharge. It my be more profitable considering that evening prices are ussually higher than morning ones. Splitting it into two steps however, secures energy for occassional evening/night activities, like washing, playing PC games till the morning etc. It creates implicit buffer. 

Energy deficit might be also caused by a bad weather next day. No enough sun will turn into empty battery for next evening. In such a case it's better to hold energy in battery to use it for household operations, rather than sell.

To predict the PV production, I'm using data proivided by Solcast integration. It provides forecast of energy to be produced by FV plant. The sensors provide total daily energy of current and incoming days. Sensors attributes contain hourly break-down of this power. It's possible optimize PV production down to half an hour granularity. But I think the daily prediction prediction is suitable enough.

The automation uses tomorrow's prediction before evening discharge, and todays prediction for morning discharge. Discharge is triggered only if forecasted energy value is greater than set one. In my case it's 20kWh.

> Selling energy when forecasted energy is not enough to cover household needs might make sense only if selling price is higher than purchase one. It happens from time to time, but not often nor regular enough to consider it common.

**Charge**

Charge during cheapest hours requires no special mode. Just General mode providing normal operation.

**Delayed Charge**

The tricky part is the time period between morning discharge and the cheapest charge. During this period the system should operate simmilarily to General mode, but it should not charge battery from the FV panels. Instead, any excess of PV energy should be injected to the grid.
Such a mode might be called `Feed-in Mode`. If not available, the inverter may provide an option to limit a charge current. Note that two methods differs a bit. Feed-in mode allows charging the battery if PV production is higher than what is possible/allowed to export to the grid. In practice it might be neglible, but it's good to be aware about it.

> Wattsonic gen3 with firmware 2.x has no Feed-in mode. This mode has been introduced in newer firmware versions.

It's possible that during this time period, spot price drops down to the level, that selling energy is not profitable any more. In such case, it's better to redirect PV energy to battery, not waiting for cheapest time window. It allows to charge (a bit) battery as soon as possible, to cover eventual  intensive usage by household.

**Optimal time windows**

The last but not least, we need to calculate time windows for charge and discharge. The time windows depend mainly on amount of energy to convert as well as velocity of this process. In real live the velocity is imnflucenced by factors like temporary household consumption or momental power. But eventual deviations are not significant enough to spend more time on them. For the same reason, the calculation is done roughly, with 15-minute granularity. 
The result is the start of the time window. Process ends when reaching predetermined SOC levels.

Let's list all sensors and configuratoin needed to adapt these automation. Values in paratheses are what's valid in my system

## The package
The whole solution is provided as a Home Assistant package. All objects created by the package are prefixed with `pv_ctrl`.

The `pv_ctrl_executor` automation is the heart of the solution. To control an inverter, it calls a `pv_ctrl` script, that implements inverter specific calls. 
Both must be edited if you want to run it with other inverter than Wattsonic/Solinteg.

The package creates following entities:

**Configuration entities**
- `input_boolean.pv_ctrl_active` - Provides option to enable or disables the automation
- `input_select.pv_ctrl_phase` - it carries current phase of the automation. Might be shown on a dashboard, or changed manually if needed for some reason.
- `input_number.pv_ctrl_min_suncast_tomorrow` - min amount of energy forecasted for following day, to allow the discharge. Used by both: evening and morning process.
- `input_number.pv_ctrl_soc_limit_morning` - the morning discharge will stop reaching this SOC. (25%)
- `input_number.pv_ctrl_soc_limit_evening` - the evening discharge will stop reaching this SOC. (70%)
- `input_number.pv_ctrl_min_export_price` - (in currency for 1MWh). We export energy only if the price is higher than defined value. It could be 0 for spot only, but eventual handling fee calculating from the volume raises this limit. If the fee is 250CZK for 1MWh, enter 250. (250)

**Template sensors**

- `sensor.pv_ctrl_most_expensive_h_morning` - template sensor, provides start time of morning discharge
- `sensor.pv_ctrl_most_expensive_h_afternoon` - template sensor, provides start time of evening discharge
- `sensor.pv_ctrl_cheapest_h_start` - template sensor, provides start time cheapest hours window, to start charging.

### External entities

The automation make use of following entities, that come from other integrations:

**Solcast entities**

- `sensor.solcast_pv_forecast_forecast_today` - PV Energy predicted for today
- `sensor.solcast_pv_forecast_forecast_tomorrow` - PV Energy predicted for tomorrow

**Spot prices**

- `sensor.current_spot_electricity_price_15min` - current spot price

**Inverter entities**
- `sensor.wattsonic_battery_soc` - for tracking current State Of Charge.

# Dashboard

![The Dashboard](images/dashboard.png)

The dashboard is matter of personal choice. This one was created to observe relationship between various values or control the inverter in case the Automation did something unexpected (happens during development).

Configuration entities are protected by Edit Mode switch. Mainly it prevents against accidental settings changes. I can confirm, it's very easy to change a slider value when scrilling the page up/down on mobile phone.

The three buttons on the top allows to activate the inverter's modes. The five buttons below, represents phases of the automation.

THe dashboard presented above uses:
- custom:apex-charts for graphs
- custom:button-card for buttons
- custom:restriction-card for locking controls