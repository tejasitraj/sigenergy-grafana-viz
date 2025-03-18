# Visualisation of Sigenergy Performance in Grafana
Sigenergy's Solar Inverter and Battery Energy Storage System (ESS) is increasingly popular in the market. Here is a good way to visualise it's performance in Grafana, real time. 

![image](https://github.com/user-attachments/assets/bc90773f-f1a0-4dc2-81e8-30e5e9921b73)

The dashboard focuses on the benefit that the system as a whole brings - i.e. compares the cost from the grid to a "would be" scenario where the system wasn't in place. It also looks at the total solar energy generated, total electricity bought from the grid and battery charge/discharge totals.

# Data Flow

![image](https://github.com/user-attachments/assets/174b649d-ea0a-4513-880a-a5bb16330b9d)

You first need to make sure that your installer enables the modbus TCP server on your Sigenergy inverter. Then, I recommend referring to [this repository](https://github.com/TypQxQ/Sigenergy-Home-Assistant-Integration) to get data into Home Assistant. It is awesome - you get over 100 modbus parameters, plus some calculated template entities!

I also then assume that you have an InfluxDB instance available, either as a Home Assistant addon or a standalone install. I use InfluxDB v2.xx, which uses the Flux Query Language. This isn't widely used; InfluxDB v1.xx is far more popular - but I don't use it! 

Finally, you should have an instance of Grafana running and connected. There are loads of videos on the internet on setting up InfluxDB and Grafana with Home Assistant. 

# The data that we will need

One major issue is that Sigenergy does not actually provide a parameter for the Solar Energy Generated. The repository linked above calculates this using an integral on the instaneous solar power, but this has been very inaccurate for me - since the power value only updates every 10 seconds at best, the data isn't granuar enough to perform an accurate integral. 

But never mind, we will calculate the Solar Energy Generated from other available parameters.

![image](https://github.com/user-attachments/assets/22ebc303-4ee6-40fe-bf4c-b8371d26d84a)

The (rather amateur) schematic above shows a simple view of energy flow thoughout the system. These are the parameters that we will be using.

- ```sensor.sigen_accumulated_export_energy``` (Modbus Parameter Address 30556): This is a sensor that shows the accumulated energy exported from the Inverter.
- ```sensor.sigen_accumulated_import_energy``` (Modbus Parameter Address 30562): This is a sensor that shows the accumulated energy imported by the Inverter - from the grid.
- ```sensor.sigen_accumulated_charge_energy``` (Modbus Parameter Address 30568): This is a sensor that shows the accumulated energy that goes into the battery
- ```sensor.sigen_accumulated_discharge_energy``` (Modbus Parameter Address 30574): This is a sensor that shows the accumulated energy that comes out of the battery

Then, we also need two sensors that show the energy imported from the grid, and energy exported from the grid. You don't get these sensors from the Sigenergy Inverter - there is no modbus parameter address for them (The Home Assistant integration linked above does have sensors for these, but again, these are based on integrals of instantaneous power, which are just not accurate for me). I have an Home Electricity Meter with a HAN port (P1 port in some countries), and I use a solution like [this](https://www.zuidwijk.com/product/slimmelezer-plus/) or [this](https://www.amsleser.no/) to read data from it. This provides the following sensors: 

- ```sensor.house_total_energy```: This is a sensor that shows the accumulated energy imported from the grid.
- ```sensor.house_total_active_export```: This is a sensor that shows the accumulated energy exported to the grid.

Finally, let's talk money - I have an electricity contract from the Swedish company Tibber, that gives an hourly price. This hourly price is in a sensor called ```sensor.torpavagen_electricity_price``` for me.

Now, configure your Home Assistant's ```configuration.yaml``` to send all these sensors to InfluxDB.

# Calculating Solar Energy and House Consumption

With the sensors above, we are now ready to calculate the data that we don't yet have.

## Solar Generated Energy

Refer to the schematic above, and focus on the inverter. We will now use a classic energy logic - **what goes into the inverter must be equal to what comes out of the inverter.**

Hence: 

```Accumulated Import Energy``` + ```Accumulated Solar (PV) Generated Energy``` + ```Accumulated Discharge Energy``` = ```Accumulated Export Energy``` + ```Accumulated Charge Energy```

Which gives us:
```Accumulated Solar (PV) Generated Energy``` = ```Accumulated Export Energy``` - ```Accumulated Import Energy``` + ```Accumulated Charge Energy``` + ```Accumulated Discharge Energy```

## House Consumption 

Now we will calculate what the house is actually consuming. Now, focus on the electricity meter in the schematic above, and use the same energy balance logic:

```Accumulated Grid Import Energy``` + ```Accumulated Export Energy (from inverter)``` = ```Accumulated Grid Export Energy``` + ```Accumulated Import Energy (to inverter)``` + ```House Consumption```

# Building the graphics
I have added the FLux code for all cards in this graphic in this repository. Feel free to use and modify as appropriate to your installation. 

# Next steps
- Add a visual showing the revenue from exported energy
- Use influxDB to create a custom sensor in home assistant for Solar Generated Energy - so it can be used in the energy dashboard (life would have been so much easier if Sigenergy just provided this datapoint on their modbus spec)
- Use a time-series database that uses an SQL-like language instead. I got "locked in" to flux pretty early on; although it is not really commonly used in the home automation community (i think). Flux has its advantages, but some real downsides. Even InfluxData have acknowledged that Flux isn't everyone's cup of tea, so they have put it in "maintenance mode"

Which gives us:

```House Consumption``` = ```Accumulated Grid Import Energy``` - ```Accumulated Grid Export Energy``` + ```Accumulated Export Energy (from inverter)``` - ```Accumulated Import Energy (to inverter)```
