---
title: "Roborock S4 Lovelace Dashboard"
excerpt: >
  This post documents my Lovelace dashboard that I've created around my
  Roborock S4 vacuum in Home Assistant.  I also go through how I've created
  each of the cards.
categories:
  - home automation
tags:
  - vacuum
  - homeassistant
  - home automation
  - roborock
header:
  image: /assets/images/0009_header.png
  image_description: Lovelace Roborock Dashboard
  caption: "Lovelace Roborock Dashboard"
classes: wide
---
*Note: this post was written about the Roborock S4 but it should also work with the S5, S5 Max, S6 and S6 Pure.*
{: .notice--warning}

<div><p>This is part 3 of a 3 part post on the vacuum.</p>
<ul>
  <li><a href="https://aarongodfrey.dev/home%20automation/integrating_the_roborock_s4_in_home_assistant/">Part 1 - Integrating the Roborock S4 in Home Assistant</a></li>
  <li><a href="https://aarongodfrey.dev/home%20automation/roborock_s4_home_assistant_automations/">Part 2 - Roborock S4 Automations and Scripts in Home Assistant</a></li>
  <li>Part 3 - Roborock S4 Lovelace Dashboard (Reading Now!)</li>
</ul>
</div>
{: .notice--info}

In this final post I will document the cards I've created for my roborock lovelace
dashboard.  I will briefly go over what each card does, any setup required and finally
the lovelace yaml to reproduce the card in your own dashboard.

## Vacuum State/Control Card

[![Vacuum State/Control Card](/assets/images/0007_lovelace_vacuum_card.png)](/assets/images/0007_lovelace_vacuum_card.png){: .align-right}
The vacuum state/control card provides some stats about the vacuum, information
about it's state, and a few common controls.  The bottom of the card contains the
following (from left to right):

* Current status (clicking opens up more details)
* Battery Icon based on current state
* Battery Percent (clicking opens up battery history)
* Locate Vacuum (clicking will call the vacuum locate service)
* Return to Dock (clicking will call the return_to_base service)
* Clean First Floor (clicking will initiate the [script to clean the first floor](https://aarongodfrey.dev/home%20automation/roborock_s4_home_assistant_automations/#vacuum-first-floor))

Before I detail the yaml necessary to create the card we will first need to create
a few custom template sensors to extract some device state attributes that will
be added in our `picture-elements` card.

```yaml
- platform: template
  sensors:

    roborock_s4_battery:
      friendly_name: 'Roborock S4 Battery'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'battery_level')}}"{% endraw %}
      unit_of_measurement: '%'
      device_class: battery

    roborock_s4_lifetime_cleaned_area:
      friendly_name: 'Lifetime Cleaned Area'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'total_cleaned_area')}}"{% endraw %}
      unit_of_measurement: ㎡

    roborock_s4_lifetime_cleaning_time:
      friendly_name: 'Lifetime Cleaning Time'
      value_template: {% raw %}"{{(state_attr('vacuum.roborock_s4', 'total_cleaning_time') / 60)|round(1, 'floor')}}"{% endraw %}

    # NOTE: This date is converted to be timezone aware so that it plays nice
    # with some other templating functions and filters.
    roborock_s4_last_cleaned:
      friendly_name: Relative time since last cleaning ended
      value_template: {% raw %}"{{relative_time(strptime(as_timestamp(state_attr('vacuum.roborock_s4', 'clean_stop'))|timestamp_custom('%Y-%m-%d %H:%M:%S%z'), '%Y-%m-%d %H:%M:%S%z'))}}"{% endraw %}

    roborock_s4_lifetime_cleaning_count:
      friendly_name: 'Lifetime Cleaning Count'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'cleaning_count')}}"{% endraw %}
```

After the above template sensors are added to your `configuration.yaml` and
Home Assistant is restarted you will see these in the States tab in the developer
tools.

This card also uses one custom card, [lovelace-card-templater](https://github.com/gadgetchnnel/lovelace-card-templater).
This can be installed manually or via [HACS](https://hacs.xyz/).

```yaml
type: 'custom:card-templater'
entities:
  - vacuum.roborock_s4
card:
  type: picture-elements
  image: "/local/roborock_s4.jpg"
  elements:
    - type: state-label
      entity: vacuum.roborock_s4
      style:
        left: 0
        right: 0
        bottom: 0
        background-color: "rgba(0, 0, 0, 0.3)"
        padding: 10px
        font-size: 16px
        line-height: 16px
        color: white
        transform: translate(0%,0%)
    - type: icon
      title: Battery
      icon_template: {% raw %}'{{ state_attr("vacuum.roborock_s4", "battery_icon") }}'{% endraw %}
      style:
        right: 171px
        bottom: 0
        padding: 10px
        font-size: 16px
        line-height: 16px
        color: white
        transform: translate(0%,0%)
    - type: state-label
      title: Battery Level
      entity: sensor.roborock_s4_battery
      style:
        right: 120px
        bottom: 0px
        padding: 10px
        font-size: 16px
        line-height: 16px
        color: white
        transform: translate(0%,0%)
    - type: icon
      title: Locate Vacuum
      icon: 'mdi:map-marker'
      style:
        right: 90px
        bottom: 0
        padding: 10px
        font-size: 16px
        line-height: 16px
        color: white
        transform: translate(0%,0%)
    - type: icon
      title: Clean First Floor
      icon: 'mdi:broom'
      tap_action:
        action: call-service
        service: mqtt.publish
        service_data:
          payload: 1
          topic: /nodered/vacuum_first_floor
      style:
        right: 0
        bottom: 0
        padding: 10px
        font-size: 16px
        line-height: 16px
        color: white
        transform: translate(0%,0%)
    - type: icon
      title: Return To Base
      icon: 'mdi:home-map-marker'
      tap_action:
        action: call-service
        service: vacuum.return_to_base
        service_data:
          entity_id: vacuum.roborock_s4
      style:
        right: 45px
        bottom: 0
        padding: 10px
        font-size: 16px
        line-height: 16px
        color: white
        transform: translate(0%,0%)
    - type: state-label
      entity: sensor.roborock_s4_last_cleaned
      prefix: 'Last ran '
      suffix: ' ago'
      style:
        background-color: "rgba(0, 0, 0, 0.3)"
        bottom: 85%
        padding: 8px
        font-size: 16px
        line-height: 2px
        color: white
        transform: translate(0%,0%)
    - type: state-label
      prefix: "Ran "
      suffix: "x"
      entity: sensor.roborock_s4_lifetime_cleaning_count
      style:
        background-color: "rgba(0, 0, 0, 0.3)"
        bottom: 85%
        right: 0px
        padding: 8px
        font-size: 16px
        line-height: 2px
        color: white
        transform: translate(0%,0%)
        width: 25%
    - type: state-label
      prefix: "Ran for "
      suffix: " hrs"
      entity: sensor.roborock_s4_lifetime_cleaning_time
      style:
        background-color: "rgba(0, 0, 0, 0.3)"
        bottom: 72%
        right: 0px
        padding: 8px
        font-size: 16px
        line-height: 2px
        color: white
        transform: translate(0%,0%)
        width: 25%
    - type: state-label
      entity: sensor.roborock_s4_lifetime_cleaned_area
      prefix: "Cleaned "
      style:
        background-color: "rgba(0, 0, 0, 0.3)"
        bottom: 59%
        right: 0px
        padding: 8px
        font-size: 16px
        line-height: 2px
        color: white
        transform: translate(0%,0%)
        width: 25%
```
The asset image for the card can be [downloaded here](/assets/images/0009_roborock_s4.jpg).
## Vacuum Room Card

[![Vacuum Room Card](/assets/images/0007_lovelace_vacuum_room_card.png)](/assets/images/0007_lovelace_vacuum_room_card.png){: .align-right}
The vacuum room card allows you to specify a particular room along with the
number of times to clean it.  In order to use this card you will first have to
obtain the coordinates for each of your rooms.  You can read how I obtained these
values [in this post](https://aarongodfrey.dev/home%20automation/integrating_the_roborock_s4_in_home_assistant/#obtaining-parameters-for-zones),
the [Home Assistant documentation](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#retrieving-zoned-cleaning-coordinates) also contains alternative instructions.

This card also requires 2 `input_select`'s to be created before it can be used.
You will need to add these to your `configuration.yaml` and restart Home Assistant (
or if you have the latest version of Home Assistant you can call the `input_select.reload`
service).

```yaml
input_select:
  vacuum_room_select:
    name: Choose a room to clean
    options:
      - Bathroom
      - Dining Room
      - Guest Bedroom
      - Hallway
      - Kitchen
      - Living Room

  vacuum_room_repeat:
    name: Number of times to clean the room
    options:
      - 1
      - 2
      - 3
    initial: 1
    icon: mdi:numeric
```

This card also uses one custom card, [lovelace-card-templater](https://github.com/gadgetchnnel/lovelace-card-templater).
This can be installed manually or via [HACS](https://hacs.xyz/).

```yaml
type: 'custom:card-templater'
card:
  type: entities
  title: Vacuum a Room
  show_header_toggle: false
  entities:
    - input_select.vacuum_room_select
    - input_select.vacuum_room_repeat
    - type: call-service
      service: mqtt.publish
      service_data:
        topic: /nodered/vacuum_room
        payload_template: >-
          {% raw %}{
            "room": "{{ states.input_select.vacuum_room_select.state }}",
            "repeat": {{ states.input_select.vacuum_room_repeat.state }}
          }{% endraw %}
      name: " "
      icon: " "
entities:
  - input_select.vacuum_room_select
  - input_select.vacuum_room_repeat
```

When the `Run` button is clicked it will publish an mqtt message to the `/nodered/vacuum_room`
topic.  This will be read by the [Vacuum Room Node-RED flow](https://aarongodfrey.dev/home%20automation/roborock_s4_home_assistant_automations/#vacuum-room), which will properly create
the parameters and make the service call.

## Maintenance Cards

[![Maintenance Cards](/assets/images/0007_lovelace_maintenance_cards.png)](/assets/images/0007_lovelace_maintenance_cards.png){: .align-right}
The maintenance cards highlight the part of the vacuum, how many hours are left
until it needs to be cleaned or replaced, and provides a way to reset the counter
after you've done the maintenance.

Before we can use any of these cards we will need to create 2 custom template
sensors for each card.  The first will create a sensor for the percentage remaining
and the second will create a sensor for the number of hours remaining.

```yaml
- platform: template
  sensors:

    # Used for the Filter maintenance card
    roborock_s4_filter_remaining:
      friendly_name: '% Filter Remaining'
      unit_of_measurement: '%'
      value_template: {% raw %}"{{((state_attr('vacuum.roborock_s4', 'filter_left') / 150) * 100) | round | int}}"{% endraw %}

    # Used for the Filter maintenance card
    roborock_s4_filter_hrs_remaining:
      friendly_name: 'Filter Remaining Hours'
      unit_of_measurement: 'hrs'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'filter_left')}}"{% endraw %}

    # Used for the Side Brush maintenance card
    roborock_s4_side_brush_remaining:
      friendly_name: '% Side Brush Remaining'
      unit_of_measurement: '%'
      value_template: {% raw %}"{{((state_attr('vacuum.roborock_s4', 'side_brush_left') / 200) * 100) | round | int}}"{% endraw %}

    # Used for the Side Brush maintenance card
    roborock_s4_side_brush_hrs_remaining:
      friendly_name: 'Side Brush Remaining Hours'
      unit_of_measurement: 'hrs'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'side_brush_left')}}"{% endraw %}

    # Used for the Main Brush maintenance card
    roborock_s4_main_brush_remaining:
      friendly_name: '% Main Brush Remaining'
      unit_of_measurement: '%'
      value_template: {% raw %}"{{((state_attr('vacuum.roborock_s4', 'main_brush_left') / 300) * 100) | round | int}}"{% endraw %}

    # Used for the Main Brush maintenance card
    roborock_s4_main_brush_hrs_remaining:
      friendly_name: 'Main Brush Remaining Hours'
      unit_of_measurement: 'hrs'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'main_brush_left')}}"{% endraw %}

    # Used for the Sensors maintenance card
    roborock_s4_sensors_remaining:
      friendly_name: '% Sensors Remaining'
      unit_of_measurement: '%'
      value_template: {% raw %}"{{((state_attr('vacuum.roborock_s4', 'sensor_dirty_left') / 30) * 100) | round | int}}"{% endraw %}

    # Used for the Sensors maintenance card
    roborock_s4_sensors_hrs_remaining:
      friendly_name: 'Sensors Remaining Hours'
      unit_of_measurement: 'hrs'
      value_template: {% raw %}"{{state_attr('vacuum.roborock_s4', 'sensor_dirty_left')}}"{% endraw %}
```

After these are added to your `configuration.yaml` and you have restarted Home
Assistant you can then add the following `picture-elements` cards to your lovelace
configuration.

```yaml
- type: horizontal-stack
  cards:
    - type: picture-elements
      image: /local/side_brush.png
      elements:
        - type: state-label
          entity: sensor.roborock_s4_side_brush_remaining
          title: '% Remaining Until Side Brush Should Be Replaced'
          style:
            font-size: 30px
            color: orange
            left: 0px
            right: 0px
            bottom: 0px
            background-color: "rgba(0, 0, 0, 0.3)"
            transform: translate(0%,0%)
        - type: state-label
          title: Hours Remaining Until Side Brush Should Be Replaced
          entity: sensor.roborock_s4_side_brush_hrs_remaining
          suffix: ' left'
          style:
            right: 0px
            bottom: 0px
            padding: 10px
            font-size: 16px
            line-height: 16px
            color: white
            transform: translate(0%,0%)
        - type: icon
          icon: mdi:restart
          title: Reset Hours
          tap_action:
            action: call-service
            service: vacuum.send_command
            service_data:
              entity_id: vacuum.roborock_s4
              command: reset_consumable
              params: ['side_brush_work_time']
            confirmation:
              text: Are you sure you want to reset the hours remaining counter for replacing the side brush?
          style:
            top: 0px
            right: 0px
            padding: 7px
            transform: translate(0%,0%)
    - type: picture-elements
      image: /local/sensors.png
      elements:
        - type: state-label
          entity: sensor.roborock_s4_sensors_remaining
          title:  "% Remaining Until Sensors Should Be Cleaned"
          style:
            font-size: 30px
            color: orange
            left: 0px
            right: 0px
            bottom: 0px
            background-color: "rgba(0, 0, 0, 0.3)"
            transform: translate(0%,0%)
        - type: state-label
          title:  "Hours Remaining Until Sensors Should Be Cleaned"
          entity: sensor.roborock_s4_sensors_hrs_remaining
          suffix: ' left'
          style:
            right: 0px
            bottom: 0px
            padding: 10px
            font-size: 16px
            line-height: 16px
            color: white
            transform: translate(0%,0%)
        - type: icon
          icon: mdi:restart
          title: Reset Hours
          tap_action:
            action: call-service
            service: vacuum.send_command
            service_data:
              entity_id: vacuum.roborock_s4
              command: reset_consumable
              params: ['sensor_dirty_time']
            confirmation:
              text: Are you sure you want to reset the hours remaining counter for cleaning the sensors?
          style:
            top: 0px
            right: 0px
            padding: 7px
            transform: translate(0%,0%)

- type: horizontal-stack
  cards:
    - type: picture-elements
      image: /local/filter.png
      elements:
        - type: state-label
          title: '% Remaining Until Filter Should Be Replaced'
          entity: sensor.roborock_s4_filter_remaining
          style:
            font-size: 30px
            color: orange
            left: 0px
            right: 0px
            bottom: 0px
            background-color: "rgba(0, 0, 0, 0.3)"
            transform: translate(0%,0%)
        - type: state-label
          title: 'Hours Remaining Until Filter Should Be Replaced'
          entity: sensor.roborock_s4_filter_hrs_remaining
          suffix: ' left'
          style:
            right: 0px
            bottom: 0px
            padding: 10px
            font-size: 16px
            line-height: 16px
            color: white
            transform: translate(0%,0%)
        - type: icon
          icon: mdi:restart
          title: Reset Hours
          tap_action:
            action: call-service
            service: vacuum.send_command
            service_data:
              entity_id: vacuum.roborock_s4
              command: reset_consumable
              params: ['filter_work_time']
            confirmation:
              text: Are you sure you want to reset the hours remaining counter for replacing the filter?
          style:
            top: 0px
            right: 0px
            padding: 7px
            transform: translate(0%,0%)
    - type: picture-elements
      image: /local/main_brush.png
      elements:
        - type: state-label
          title: '% Remaining Until Main Brush Should Be Replaced'
          entity: sensor.roborock_s4_main_brush_remaining
          style:
            font-size: 30px
            color: orange
            left: 0px
            right: 0px
            bottom: 0px
            background-color: "rgba(0, 0, 0, 0.3)"
            transform: translate(0%,0%)
        - type: state-label
          title: 'Hours Remaining Until Main Brush Should Be Replaced'
          entity: sensor.roborock_s4_main_brush_hrs_remaining
          suffix: ' left'
          style:
            right: 0px
            bottom: 0px
            padding: 10px
            font-size: 16px
            line-height: 16px
            color: white
            transform: translate(0%,0%)
        - type: icon
          icon: mdi:restart
          title: Reset Hours
          tap_action:
            action: call-service
            service: vacuum.send_command
            service_data:
              entity_id: vacuum.roborock_s4
              command: reset_consumable
              params: ['main_brush_work_time']
            confirmation:
              text: Are you sure you want to reset the hours remaining counter for replacing the main brush?
          style:
            top: 0px
            right: 0px
            padding: 7px
            transform: translate(0%,0%)
```
Clicking on the reset icon will pop up a confirmation alert that you will have to
click on to continue with the action.  This was added to prevent an accidental
click on the card from reseting the counter.

The assets for the above cards can be found below:
* [filter.png](/assets/images/0009_filter.png)
* [main_brush.png](/assets/images/0009_main_brush.png)
* [side_brush.png](/assets/images/0009_side_brush.png)
* [sensors.png](/assets/images/0009_sensors.png)
