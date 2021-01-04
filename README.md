# RAPL-visualize
Visualize real-time RAPL (Running Average Power Limit) usage using Home Assistant (running on Ubuntu Linux) and Microsoft Power BI

## About
RAPL is a feature for Intel-based processors introduced on the Sandy Bridge microarchitecture, which allows for several power-related features. In this case we are using the power metering capabilities, but there are [multiple other features](https://01.org/blogs/2014/running-average-power-limit-%E2%80%93-rapl) that can be realized using RAPL.

The main purpose of this project is to show the real-time capabilities of [Power BI streaming data sets](https://docs.microsoft.com/en-us/power-bi/connect-data/service-real-time-streaming) and how simple it can be to hook into a push data set's REST API using a platform such as Home Assistant. This set up could be configured on any Linux platform easily (and instructions are even included in Power BI to configure this directly after creation of a streaming dataset). Using Home Assistant gives us a more declarative YAML-based syntax and the power of Jinja templates to easily format our request and perform minor transformations to the input data unit (µJ). In addition, since many other sensors would likely be in your Home Assistant, it can be useful to pass them along to Power BI; in this case I also pass sensors for the CPU temperature and the temperature for the room in which the server resides.

## Configuration

According to the [power capping framework](https://www.kernel.org/doc/html/latest/power/powercap/powercap.html) we can begin to locate the data in the sysfs tree: `/sys/class/powercap/intel-rapl/`. This setup is best quoted from with the framework link from above:

> There is one control type called intel-rapl which contains two power zones, intel-rapl:0 and intel-rapl:1, representing CPU packages. Each of these power zones contains two subzones, intel-rapl:j:0 and intel-rapl:j:1 (j = 0, 1), representing the “core” and the “uncore” parts of the given CPU package, respectively. All of the zones and subzones contain energy monitoring attributes (energy_uj, max_energy_range_uj) and constraint attributes (constraint_*) allowing controls to be applied (the constraints in the ‘package’ power zones apply to the whole CPU packages and the subzone constraints only apply to the respective parts of the given package individually). Since Intel RAPL doesn’t provide instantaneous power value, there is no power_uw attribute.



### Create Home Assistant Sensors

In order to place these as sensors in Home Assistant, we setup the following:
* [Command Line Sensor](https://www.home-assistant.io/integrations/sensor.command_line/) for each RAPL, including the package, core, and uncore values.
* [Statistics Sensor](https://www.home-assistant.io/integrations/statistics/) which we utilize to get the average change over a specified duration (in this case we just use the default of 20 samples)

Quick Tip: You can confirm the name of the interface for RAPL using the following command:
```
cat /sys/class/powercap/intel-rapl/intel-rapl:0/name
```
Please note that the framework uses `power_cap` but my system uses `powercap` (no underscore)!

Based off what was outlined above, we get the following YAML which can be placed directly into your `configuration.yaml`:
```
  - platform: statistics
    entity_id: sensor.rapl_full
    name: stats_rapl_full
  - platform: statistics
    entity_id: sensor.rapl_core
    name: stats_rapl_core
  - platform: statistics
    entity_id: sensor.rapl_uncore
    name: stats_rapl_uncore
  - platform: command_line
    name: RAPL - full
    command: 'cat /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj'
    scan_interval: 1
    value_template: '{{ value | multiply(0.000001) | round(1) }}'
    unit_of_measurement: 'J'
  - platform: command_line
    name: RAPL - core
    command: 'cat /sys/class/powercap/intel-rapl/intel-rapl:0/intel-rapl:0:0/energy_uj'
    scan_interval: 1
    value_template: '{{ value | multiply(0.000001) | round(1) }}'
    unit_of_measurement: 'J'
  - platform: command_line
    name: RAPL - uncore
    command: 'cat /sys/class/powercap/intel-rapl/intel-rapl:0/intel-rapl:0:1/energy_uj'
    scan_interval: 1
    value_template: '{{ value | multiply(0.000001) | round(1) }}'
    unit_of_measurement: 'J'

 # ------ Optional CPU temp sensor

  - platform: command_line
    name: CPU Temperature
    command: "cat /sys/class/thermal/thermal_zone0/temp"
    # If errors occur, make sure configuration file is encoded as UTF-8
    unit_of_measurement: "°C"
    value_template: '{{ value | multiply(0.001) | round(1) }}'
```

The only thing that needs to be explained is that we use the `value_template` key to convert the value from µJ to J for each of the sensors, and then round it to a single digit (you can choose your preference here). Since you can use multiple different interfaces to access a sensor like CPU temp in Home Assistant (e.g. [Glances add-on](https://www.home-assistant.io/integrations/glances/) I just added a simple optional sensor which seems to work well on my platform.

## Create Power BI Streaming Push Data Set

After having this configuration, we need to set up Power BI with a streaming data set. I won't go into detail here about how to login to Power BI, but if you're having issues you can follow [this tutorial](https://youtu.be/uZyy_qqRPiU?t=337) to create an AAD tenant with a user and login with that user at https://app.powerbi.com

Once logged in, click on your workspace and then create a new dashboard:
