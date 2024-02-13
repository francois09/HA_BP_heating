# homeassistant

This repository contains blueprints for heating and light. As you can imagine, `heater_*` are for heating management,
and `light_*` are for ... light management.

## Principle

### Heating system

1. Each room is autonomous, and have helpers for their properties
2. Each room should have a climate valve, and a T° sensor
3. A global blueprint is in charge of collecting rooms needs to decide to start heater/climate
4. A global boolean (winter_mode) is in charge to determine if it's ... winter :)

### Light system

1. Each room is independant, and must have a Dimmer and a motion sensor
2. Light will depend on cloud cover, window, and sun
3. Manual override is possible

## How it works

### Heating system

Based on triggers, each room will compute the T° setpoint, and if necessary activate a helper
saying it needs heating.

The rooms are also able to determine if they're linked to inside, outside, both or none. Depending on link,
room climate valve can be set to off, or whole heating system can be stopped.


Room Link | Action taken
---|---
Inside | Nothing special (Valve reset to unlocked and started)
Outside | Valve is closed and stopped
Both | Flag setup for shutdown global heater (Valve reset to unlocked and started)
None | Nothing special (Valve reset to unlocked and started)

Another automation will decide to start or stop the heating system if rooms need it or no.

On non-winter mode, all climate valves will be set to "off" and also "child lock" mode, to preserve battery.

### Light system

TBD

## How to setup

### Heating system

1. First, you need to create global and room helpers. `RO` indicates automation don't touch value, `RW` indicates value can be
set by automations.

#### Room related helpers:

* Confort T° (RO) : Climate valve T° range values. This indicates the leaving T°
* Eco T° (RO) : Climate valve T° range values. This indicates the economic T°
* Setpoint T° (RW) : Climate valve T° range values. Automation can decide the value based on Planning
* Manual T° (RW) : Boolean indicating that Setpoint is manually determined. Set when climate valve is manually modified
* Heat (RW) : Boolean indicating that the room need heating
* Debounce (RW) : Boolean avoiding race conditions on starting/setup climate valve
* Used (RO) : Boolean indicating the room (mostly bedrooms) are subject to automations. If not, T° is always set to Eco
* Planning (RO) : Planification for switching between Confort (on) and Eco (off) T°
* Linked (RW) : List (Inside, Outside, Both, None) indicating how room is connected to the house or the outside
* Selector timer (RW) : Timed selector helper. Wait until manual T° is selected
* Regulated (RO) : Boolean indicating the room are able to request heating. If off, no heat can be request, only valve changes allowed.

#### Global helpers

* Winter mode (RO) : Boolean indicating if we are in Winter (or heating season) or not
* Room link timer (RW) : Timer used to avoid reacting too quick on a room link status change
* Always Open (RO) : Binary sensor that can be used in room link decision
* Always Closed (RO) : Binary sensor that can be used in room link decision
* Heating automation pause (RW) : Timer used to avoid boot side effects.

2. You need to instantiate `heater_switch_mode.yaml`, `heater_setpoint_compute.yaml`, `heater_room_link.yaml` and `heater_request_compute.yaml` for each
room you want to activate automation.

3. Then `heater_one_need.yaml`, `heater_start_stop.yaml` and `heater_boot_management.yaml` should also be created to globally manage heating system.

### Light system

TBD

## Blueprints

### Heater system management global script

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_one_need.yaml)

This blueprint is the global on/off heating system. It collects every heater requests from rooms, and decide if home heater should be started or not. It also check
for rooms links to determine if heating system should be shutdown.

### Heater boot management

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_boot_management.yaml)

This blueprint is used at HA boot, to avoid receiving climate valve messages for a certain time. Sometimes, setpoint is received just after the boot, turning climate to be interpreted as a manual selection.

### Heater request compute

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_request_compute.yaml)

This blueprint compute the boolean (switch) helper of the room, based on Climate valve setpoint, and room T°. If heating is requested, it aslo fake the local_temperature_calibration to ensure valve is correctly opened. In my case, T° sensor is far from Climate, and close to my preffered location in the room.

### Heater setpoint compute

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_setpoint_compute.yaml)

This blueprint compute the correct setpoint based on the room schedule and the potential manual setting, to determine and send it to the Climate valve.

### Heater switch mode

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_switch_mode.yaml)

This blueprint take into account a manual change on the Climate valve, or the Auto setting to modify setpoint.

### Heater room link management

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_room_link.yaml)

This blueprint use the doors/windows status to determine if a room is linked to Inside/Outside/Both/None. When linked Outside, it stop and lock climate valve. It also setup a link helper to decide if we have to shutdown heating system or not.

### Heater Winter mode management

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_blueprints%2Fmain%2Fheater_start_stop.yaml)

This blueprint is the winter mode heating management system. It turn off and lock every climate valve to preserve battery during non-heating period.

