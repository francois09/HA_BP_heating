# Home-Assistant central heating management

This repository contains blueprints for heating system management.

## Pre-requisits

- Having a on/off central heating system.
- Using TRV like https://www.zigbee2mqtt.io/devices/TV02-Zigbee.html#tuya-tv02-zigbee on wich you can change the mode (auto/manual) and the setpoint locally, and able to accept calibration value for T° sensor
- Using specific T° sensors on the "best place" of each room
- Using contact sensors for windows and doors on each rooms

## Principle

I focus on some major ideas for building this set of blueprints:

1. Each room should be autonomous for requesting heating, and have some properties (schedule, comfort and eco T°, active or passive, etc.)
2. I don't use TRV T° sensor to determine room T°. Too close from the radiator to be accurate.
3. I fake the calibration of TRV to be sure valve is open or closed when needed

## How it works

Based on properties, and triggered by T°, properties changes, etc. each room will compute the T° setpoint, and if necessary activate a specific helper saying it needs heating. This helper set is used as a trigger for starting or not the central heating system.

The rooms are also able to determine if they're linked to inside, outside, both or none. Depending on link situation, room climate valve can be set to off, or whole heating system can be stopped.

Room Link | Action taken
---|---
Inside | Nothing special (Valve reset to child-unlocked and on "heating" mode)
Outside | Valve is child-locked and stopped
Both | Flag setup for shutdown global heater (Valve reset to child-unlocked and on "heating" mode)
None | Nothing special (Valve reset to child-unlocked and on "heating" mode)

On non-winter mode, all climate valves will be set to "off" and also "child lock" mode, to preserve battery.

## How to setup

1. First, you need to create global and room helpers. `RO` indicates automations don't touch value, `RW` indicates value can be set by automations.

#### Room related helpers:

Name|Mode|Values|Usage
---|---|---|---
Comfort T°|RO|Climate valve T° range values | This indicates the standard comfort T° requested
Eco T°|RO|Climate valve T° range values|This indicates the economic (unused room, or night mode) T°
Planning|RO|Planification|Used for switching between Confort (on) and Eco (off) T°
Setpoint T°|RW|Climate valve T° range values|Current setpoint, manually set or based on planning
Manual T°|RW|Boolean|Indicating that Setpoint is manually determined. Set when TRV is manually modified
Heat|RW|Boolean|Indicating that the room need heating
Debounce|RW|Boolean|Avoiding race conditions on starting/setup climate valve
Used|RO|Boolean|Indicating the room (mostly bedrooms) are subject to planning. If not, T° is always set to Eco
Linked|RW|List (Inside, Outside, Both, None)|Indicating how room is connected to the house or the outside
Selector timer|RW|Timer|Wait until manual T° is selected. Manually selection send too much messages.
Active|RO|Boolean|Indicating the room is able to request heating. If off, no heat can be request, only valve changes allowed.

#### Global helpers

Name|Mode|Values|Usage
---|---|---|---
Winter mode|RO|Boolean|Indicating if we are in Winter (or heating season) or not
Room link timer|RW|Timer|Used to avoid reacting too quick on a room link status change
Always Open|RO|Binary sensor|Fake sensor that can be used in room link decision
Always Closed|RO|Binary sensor|Same as above
Heating automation pause|RW|Timer|Used to avoid boot side effects.

2. For each room you want to activate automation, you need to instantiate `Room heating mode switching` (heater_switch_mode.yaml), `Room heating setpoint compute` (heater_setpoint_compute.yaml, `Room link status` (heater_room_link.yaml) and `Room heating request compute` (heater_request_compute.yaml).

3. Then create only one instance for each of `Heating system should start` (heater_one_need.yaml), `Heating system operationnal or not` (heater_start_stop.yaml) and `Heater boot management` (heater_boot_management.yaml) to globally manage heating system.

## How to use, and how I use it.

First, you need to create the plannings for all the rooms you want to manage. For example, my bedroom have a planning for every day, saying "Comfort T° from 6:30am to 8:00am and from 9:00pm to 11:00pm". Meaning I'll go to bed and wakeup on a comfortable T°, but no heating during the night or the day.

Then you have to set the room "Active" or not. "Active" allow a room to request for heating if T° is under setpoint. In my case, the entry room is only passive. Setpoint is fixed at 19°C, so if heating system is running, the room will acquire some residual heat, but never request for specific heating.

You then can decide if the room is "Used" or not. For example, my own bedroom is "Used", but the friends bedrooms are only marked "Used" when I have people home. The living room is marked "Used" for example. "Used" means using T° scheduler for changing the setpoint T°.

When you change setpoint on the TRV, the room is marked as "Manual", and the Setpoint T° will remain as selected. You can change it to "Not manual" into Home-Assistant, or (like me), press the knob on the TRV to select "Automatic". This will turn back to non manual mode into Home-Assistant. But if you look closer, the TRV will turn back to manual. The reason is that I don't use TRV "Automatic" mode with planning, I decided to use Home-Assistant planning instead.

The "Holidays" mode on my TRV is not yet used, but will probably be to say "The room is no more used, keep it on Eco T°"

## Blueprints

### Heater system management global script

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_one_need.yaml)

This blueprint is the global on/off heating system. It collects every heater requests from rooms, and decide if home heater should be started or not. It also check
for rooms links to determine if heating system should be shutdown.

### Heater boot management

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_boot_management.yaml)

This blueprint is used at HA boot, to avoid receiving climate valve messages for a certain time. Sometimes, setpoint is received just after the boot, turning climate to be interpreted as a manual selection.

### Heater request compute

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_request_compute.yaml)

This blueprint compute the boolean (switch) helper of the room, based on Climate valve setpoint, and room T°. If heating is requested, it aslo fake the local_temperature_calibration to ensure valve is correctly opened. In my case, T° sensor is far from Climate, and close to my preffered location in the room.

### Heater setpoint compute

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_setpoint_compute.yaml)

This blueprint compute the correct setpoint based on the room schedule and the potential manual setting, to determine and send it to the Climate valve.

### Heater switch mode

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_switch_mode.yaml)

This blueprint take into account a manual change on the Climate valve, or the Auto setting to modify setpoint.

### Heater room link management

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_room_link.yaml)

This blueprint use the doors/windows status to determine if a room is linked to Inside/Outside/Both/None. When linked Outside, it stop and lock climate valve. It also setup a link helper to decide if we have to shutdown heating system or not.

### Heater Winter mode management

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffrancois09%2FHA_BP_heating%2Fblob%2Fmain%2Fheater_start_stop.yaml)

This blueprint is the winter mode heating management system. It turn off and lock every climate valve to preserve battery during non-heating period.

