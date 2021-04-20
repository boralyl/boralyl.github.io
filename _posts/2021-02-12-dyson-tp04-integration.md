---
title: "Integrating the Dyson TP04 Air Purifier in Home Assistant"
excerpt: >
  In this post I integrate my Dyson TP04 Air Purifier in Home Assistant. I'll get it
  configured, add some custom sensors and create a card to display in the dashboard.
categories:
  - home automation
tags:
  - homeassistant
  - dyson
header:
  image: /assets/images/0017_header.png
  image_description: "Comparing calculated AQI from the dyson sensor to a purple air indoor air quality sensor."
  caption: "Comparing calculated AQI from the dyson sensor to a purple air indoor air quality sensor."
toc: true
---

## Why the Dyson TP04?

I currently reside in the Pacific North West and wild fires occur in the region seasonally.
With the rise of climate change they are getting worse, so after last year's several week
stint of hazardous air quality I wanted to pick up an air purifier to help indoors.

Naturally, I was inclined to find an air purifier that I could control with Home Assistant
and would ideally operate locally without the need for the cloud. During my research the
first model that stood out was the [Xioami Mi Air Purifer 3H](https://www.mi.com/global/mi-air-purifier-3H).
Unfortunately this was not available at US retailors and was even difficult to find
on sites like GearBest and AliExpress.

I decided to scrap the idea of the Xioami Mi Air Purifier and searched for other options
in the [Home Assistant community forums](https://community.home-assistant.io/). I found
several posts discussing the [Dyson air purifiers](https://www.dyson.com/air-treatment/purifiers).
Of the models listed, Home Assistant currently supports the `DP04`, `TP04` and `PH01` models.
After looking at the 3 models I immediately ruled out the [DP04](https://www.dyson.com/air-treatment/purifiers/dyson-pure-cool/dyson-pure-cool-desk-white-silver)
as it's not longer available. The [PH01](https://www.dyson.com/air-treatment/purifier-humidifiers/dyson-pure-humidify-cool/dyson-humidify-cool-white-silver) was also ruled out based on
price and that it was also a humidifier which I didn't need. The remaining option was the
[TP04](https://www.dyson.com/air-treatment/purifiers/dyson-pure-cool/dyson-pure-cool-tower-white-silver).

When I started looking into the [TP04](https://www.dyson.com/air-treatment/purifiers/dyson-pure-cool/dyson-pure-cool-tower-white-silver)
I was initially put off by the price. It's not a cheap device by any means, but after
reading some reviews I ended up justifying the cost based on the following reasons:

- It's an air purifier - Yes, this was the original goal and even those on the cheaper side
  start at around $200.
- It has a lot of sensors - I was pretty impressed by the advertised sensors I'd be able
  to use in home assistant including:
  - Temperature
  - Humidity
  - AQI
  - Particulate Matter 2.5
  - Particulate Matter 10
  - NO2
  - Volatile Organic Compounds
- A stand alone indoor air sensor from [purple air](https://www2.purpleair.com/collections/air-quality-sensors/products/purpleair-pa-i-indoor) already runs at about $200. So
  combining that with the air purifier would put it at about $400 without any "smart features".
- It has really good smart features - The Auto mode is a "set it and forget it" mode that will
  automatically purify the air. When you pair that with schedules and night mode, which will
  make it operate at quietly as possible, you have some really useful features.
- Local control via MQTT - While a network request is used to fetch registered devices once
  during the integration setup, the rest of the sensor data and control of the devices is
  done via MQTT locally.
- It doubles as a fan - This is super minor and probably contributes least to why I
  decided to go with this model.

## Integration

<div>
<strong>Update 20201-04-19:</strong>

<p>
A few months ago, the official integration broke due to an external dependency that is not
being maintained. Dyson made some updates and added 2FA which broke the external dependency.
</p>
<p>
I now recommend that you use the <a href="https://github.com/shenxn/ha-dyson">ha-dyson</a> custom
integration that can be installed via HACS. It not only works, but also can be used 100%
locally after the initial setup to gather authentication data for you devices from the cloud.
</p>
<p>
I'm leaving the rest of the original instructions this section here for posterity.
</p>
</div>
{: .notice--warning}

After installing the Dyson Link app and connecting the air purifier, I added the necessary
[configuration](https://www.home-assistant.io/integrations/dyson/) to my `configuration.yaml`.

```yaml
# Enables Dyson integration (for our air purifier (Dyson pure cool TP04)
# https://www.home-assistant.io/integrations/dyson/
dyson:
  username: !secret dyson_username
  password: !secret dyson_password
  language: US
  devices:
    device_id: !secret dyson_id
    device_ip: !secret dyson_ip
```

After a restart I had the following new entities:

- `air_quality.bedroom`
- `fan.bedroom`
- `sensor.bedroom_humidity`
- `sensor.bedroom_temperature`
- `sensor.bedroom_hepa_filter_remaining_life`
- `sensor.bedroom_carbon_filter_remaining_life`

This integration has all the normal `fan` services and a few that are unique to the device.
As of writing the following additional services are available for this model:

- [dyson.set_speed](https://www.home-assistant.io/integrations/dyson/#service-dysonset_speed) - Set the exact speed (1-10) of the fan.
- [dyson.set_auto_mode](https://www.home-assistant.io/integrations/dyson/#service-dysonset_auto_mode) - Toggle the fan's auto mode.
- [dyson.set_night_mode](https://www.home-assistant.io/integrations/dyson/#service-dysonset_night_mode) - Toggle the fan's night mode.
- [dyson.set_angle](https://www.home-assistant.io/integrations/dyson/#service-dysonset_angle-only-for-dp04-and-tp04) - Set the oscillation angle of the fan.
- [dyson.set_flow_direction_front](https://www.home-assistant.io/integrations/dyson/#service-dysonset_flow_direction_front-only-for-dp04-and-tp04) - Set the flow direction of the fan.
- [dyson.set_timer](https://www.home-assistant.io/integrations/dyson/#service-dysonset_timer-only-for-dp04-and-tp04) - Set a sleep timer.

## Template Sensors

While there are attributes on the fan entity that is created for the various sensors,
I wanted to pull each of them out into their own sensor. My primary reason for this was
so that data could be fed to [influxdb](https://www.influxdata.com/products/influxdb/) and I could visualize it using [grafana](https://grafana.com/). I ended up creating separate sensors
for Air Quality Index, Volatile Organic Compounds, NO2, Particulate Matter 2.5, and
Particulate Matter 10. Below are the template sensors in my `configration.yaml`.

```yaml
sensor:
  - platform: template
    sensors:
      dyson_aqi:
        friendly_name: "Dyson TP-04 Air Quality Index"
        value_template: "{{ state_attr('air_quality.bedroom', 'air_quality_index') }}"
        unit_of_measurement: AQI
        unique_id: dyson_tp_04_aqi

      dyson_particulate_matter_2_5:
        friendly_name: "Dyson TP-04 PM2.5"
        value_template: "{{ state_attr('air_quality.bedroom', 'particulate_matter_2_5') }}"
        unit_of_measurement: "µg/m³"
        unique_id: dyson_tp_04_pm2_5

      dyson_particulate_matter_10:
        friendly_name: "Dyson TP-04 PM10"
        value_template: "{{ state_attr('air_quality.bedroom', 'particulate_matter_10') }}"
        unit_of_measurement: "µg/m³"
        unique_id: dyson_tp_04_pm_10

      dyson_volatile_organic_compounds:
        friendly_name: "Dyson TP-04 Volatile Organic Compounds"
        value_template: "{{ state_attr('air_quality.bedroom', 'volatile_organic_compounds') }}"
        unit_of_measurement: "VOC Scale"
        unique_id: dyson_tp_04_voc

      dyson_nitrogen_dioxide:
        friendly_name: "Dyson TP-04 Nitrogen Dioxide"
        value_template: "{{ state_attr('air_quality.bedroom', 'nitrogen_dioxide') }}"
        unit_of_measurement: NO2 Scale
        unique_id: dyson_tp_04_no2
```

## Calculating AQI (based on PM2.5)

One thing I noticed about the AQI sensor, is that the integration simply returns the maximum
number from the PM2.5, PM10, NO2 and VOC sensors as it's value. That particular value isn't
all that useful since each of those sensors have their own unit of measurement. It's not clear in
the Dyson Link app how that value is calculated and it isn't exposed in the MQTT data.

What I wanted was a number like I use for my [PurpleAir PA-I-Indoor sensor](https://www2.purpleair.com/collections/air-quality-sensors/products/purpleair-pa-i-indoor). Fortunately the formula
was pulled from the javascript code in the purple air site [in this community post](https://community.home-assistant.io/t/purpleair-air-quality-sensor/146588). Using that formula I created
my own calculated AQI based on PM2.5 as a template sensor (this can be added with the other
template sensors created above):

```yaml
- platform: template
  sensors:
    dyson_calc_aqi:
      friendly_name: "Dyson Calculated PM2.5 AQI"
      unit_of_measurement: AQI
      unique_id: dypson_tp_04_calc_aqi
      # https://community.home-assistant.io/t/purpleair-air-quality-sensor/146588
      value_template: >
        {% raw %}{% macro calcAQI(Cp, Ih, Il, BPh, BPl) -%}
          {{ (((Ih - Il)/(BPh - BPl)) * (Cp - BPl) + Il)|round }}
        {%- endmacro %}
        {% if (states("sensor.dyson_particulate_matter_2_5")|float) > 1000 %}
          invalid
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 350.5 %}
        {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 500.0, 401.0, 500.0, 350.5) }}
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 250.5 %}
         {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 250.5 %}
        {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 400.0, 301.0, 350.4, 250.5) }}
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 150.5 %}
          {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 300.0, 201.0, 250.4, 150.5) }}
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 55.5 %}
          {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 200.0, 151.0, 150.4, 55.5) }}
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 35.5 %}
          {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 150.0, 101.0, 55.4, 35.5) }}
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) > 12.1 %}
          {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 100.0, 51.0, 35.4, 12.1) }}
        {% elif (states("sensor.dyson_particulate_matter_2_5")|float) >= 0.0 %}
          {{ calcAQI((states("sensor.dyson_particulate_matter_2_5")|float), 50.0, 0.0, 12.0, 0.0) }}
        {% else %}
          invalid
        {% endif %}{% endraw %}
```

To verify that this was calculating the value correctly, I moved my PurpleAir indoor
sensor next to the Dyson air purifier for 6 hours and plotted the 2 values on a graph in
grafana.

[![Grafana Graph](/assets/images/0017_header.png)](/assets/images/0017_header.png)

After examining the graph it was pretty clear that the calculation was matching that used
by the PurpleAir indoor sensor.

## Custom Card

After I created all of the template sensors I wanted, I created a custom card to be able
to display all of the sensor data and to control the air purifier. To do this I used the
extremely versatile [picture-elements lovelace card](https://www.home-assistant.io/lovelace/picture-elements/). Before I show you the code, here is the finished card in my dashboard:

[![Dyson Air Purifier Custom Card](/assets/images/0017_dyson_card.png)](/assets/images/0017_dyson_card.png)

On the left of the card are the sensors for calculated AQI, PM2.5, PM10, NO2 and VOC.
The background color of these sensors will change based on the level reported. So they
will be green when in healthy levels than transition to orange, red, purple and burgundy
when hazordous. In the middle of the card are circular cards representing the % remaining of the HEPA
and carbon filters. On the right of the card are the current humidity and temperature
readings. Along the bottom are buttons to control the purifier. They will be green when
enabled and the default color when not enabled. From left to right they represent
On/Off, Oscillation, Air flow direction, Auto Mode and Night Mode.

In order to use this card as-is you will need to install 2 custom cards from [HACS](https://hacs.xyz/):

- [lovelace-card-templater](https://github.com/gadgetchnnel/lovelace-card-templater)
- [circle-sensor-card](https://github.com/custom-cards/circle-sensor-card)

You will also need to create the [template sensors above](#template-sensors) or modify
the yaml below to reference attributes of your `fan` entity. Obviously you will need to
make sure to update entity names referenced in the yaml to match your names. The last
thing you will need is the background image which can be [downloaded here](/assets/images/0017_card_background.png).

```yaml
- type: 'custom:card-templater'
  entities:
    - fan.bedroom
    - sensor.dyson_calc_aqi
    - sensor.dyson_volatile_organic_compounds
    - sensor.dyson_nitrogen_dioxide
    - sensor.dyson_particulate_matter_2_5
    - sensor.dyson_particulate_matter_10
  card:
    type: picture-elements
    image: "/local/card_background.png"
    elements:

      # Buttons
      {% raw %}
      - type: state-icon
        icon: mdi:power
        entity: fan.bedroom
        tap_action:
          action: call-service
          service: fan.toggle
          service_data:
            entity_id: fan.bedroom
        style:
          left: 0
          right: 0
          bottom: 0
          border: 1px solid rgba(0, 0, 0, 0.4)
          background-color: "rgba(0, 0, 0, 0.8)"
          padding: 10px
          font-size: 16px
          line-height: 16px
          transform: translate(0%,0%)
          --paper-item-icon-active-color: green

      - type: state-icon
        state_color: false
        icon: mdi:sleep
        title_template: 'Night mode {% if state_attr("fan.bedroom", "night_mode") %}(click to disable){% else %}(click to enable){% endif %}'
        entity: fan.bedroom
        tap_action:
          action: call-service
          service: dyson.set_night_mode
          service_data:
            entity_id: fan.bedroom
            night_mode_template: >-
              {% if state_attr("fan.bedroom", "night_mode") %}false{% else %}true{% endif %}
        style:
          right: 0
          bottom: 0
          padding: 10px
          transform: translate(0%,0%)
          font-size: 16px
          line-height: 16px
          --paper-item-icon-color_template: '{% if state_attr("fan.bedroom", "night_mode") %}green{% else %}rgb(68, 115, 158){% endif %}'

      - type: state-icon
        state_color: false
        icon: mdi:alpha-a-circle
        title_template: 'Auto mode {% if state_attr("fan.bedroom", "auto_mode") %}(click to disable){% else %}(click to enable){% endif %}'
        entity: fan.bedroom
        tap_action:
          action: call-service
          service: dyson.set_auto_mode
          service_data:
            entity_id: fan.bedroom
            auto_mode_template: >-
              {% if state_attr("fan.bedroom", "auto_mode") %}false{% else %}true{% endif %}
        style:
          right: 61px
          bottom: 0
          padding: 10px
          transform: translate(0%,0%)
          font-size: 16px
          line-height: 16px
          --paper-item-icon-color_template: '{% if state_attr("fan.bedroom", "auto_mode") %}green{% else %}rgb(68, 115, 158){% endif %}'

      - type: state-icon
        state_color: false
        icon: mdi:air-purifier
        title_template: 'Airflow direction {% if state_attr("fan.bedroom", "flow_direction_front") %}(click to set flow direction behind){% else %}(click to set flow direction to front){% endif %}'
        entity: fan.bedroom
        tap_action:
          action: call-service
          service: dyson.set_flow_direction_front
          service_data:
            entity_id: fan.bedroom
            flow_direction_front_template: >-
              {% if state_attr("fan.bedroom", "flow_direction_front") %}false{% else %}true{% endif %}
        style:
          right: 122px
          bottom: 0
          padding: 10px
          transform: translate(0%,0%)
          font-size: 16px
          line-height: 16px
          --paper-item-icon-color_template: '{% if state_attr("fan.bedroom", "flow_direction_front") %}green{% else %}rgb(68, 115, 158){% endif %}'

      - type: state-icon
        state_color: false
        icon: mdi:arrow-split-vertical
        title_template: 'Oscillation {% if state_attr("fan.bedroom", "oscillating") %}(click to turn off){% else %}(click to turn on){% endif %}'
        entity: fan.bedroom
        tap_action:
          action: call-service
          service: fan.oscillate
          service_data:
            entity_id: fan.bedroom
            oscillating_template: >-
              {% if state_attr("fan.bedroom", "oscillating") %}false{% else %}true{% endif %}
        style:
          right: 183px
          bottom: 0
          padding: 10px
          transform: translate(0%,0%)
          font-size: 16px
          line-height: 16px
          --paper-item-icon-color_template: '{% if state_attr("fan.bedroom", "oscillating") %}green{% else %}rgb(68, 115, 158){% endif %}'

      # Sensors

      - type: state-label
        entity: sensor.dyson_calc_aqi
        suffix: ' (PM2.5)'
        style:
          background-color_template: >-
            {% if states("sensor.dyson_calc_aqi")|float > 300 %}
            #B71C1C
            {% elif states("sensor.dyson_calc_aqi")|float > 200 %}
            #9C27B0
            {% elif states("sensor.dyson_calc_aqi")|float > 150 %}
            #E53935
            {% elif states("sensor.dyson_calc_aqi")|float > 100 %}
            #FB8C00
            {% elif states("sensor.dyson_calc_aqi")|float > 50 %}
            #FFC107
            {% else %}
            #4CAF50
            {% endif %}
          width: 25%
          top: 3%
          left: 0%
          border-top-right-radius: 7px
          border-bottom-right-radius: 7px
          border: "2px solid rgba(0, 0, 0, 0.2)"
          padding: 5px
          font-size: 16px
          line-height: 16px
          color: white
          transform: translate(0%,0%)

      - type: state-label
        entity:  sensor.dyson_particulate_matter_2_5
        suffix: ' PM2.5'
        style:
          background-color_template: >-
            {% if states("sensor.dyson_particulate_matter_2_5")|float > 250 %}
            #B71C1C
            {% elif states("sensor.dyson_particulate_matter_2_5")|float > 150 %}
            #9C27B0
            {% elif states("sensor.dyson_particulate_matter_2_5")|float > 70 %}
            #E53935
            {% elif states("sensor.dyson_particulate_matter_2_5")|float > 53 %}
            #FB8C00
            {% elif states("sensor.dyson_particulate_matter_2_5")|float > 35 %}
            #FFC107
            {% else %}
            #4CAF50
            {% endif %}
          width: 25%
          top: 17%
          left: 0%
          border-top-right-radius: 7px
          border-bottom-right-radius: 7px
          border: "2px solid rgba(0, 0, 0, 0.2)"
          padding: 5px
          font-size: 16px
          line-height: 16px
          color: white
          transform: translate(0%,0%)

      - type: state-label
        entity:  sensor.dyson_particulate_matter_10
        suffix: ' PM10'
        style:
          background-color_template: >-
            {% if states("sensor.dyson_particulate_matter_10")|float > 420 %}
            #B71C1C
            {% elif states("sensor.dyson_particulate_matter_10")|float > 350 %}
            #9C27B0
            {% elif states("sensor.dyson_particulate_matter_10")|float > 100 %}
            #E53935
            {% elif states("sensor.dyson_particulate_matter_10")|float > 75 %}
            #FB8C00
            {% elif states("sensor.dyson_particulate_matter_10")|float > 50 %}
            #FFC107
            {% else %}
            #4CAF50
            {% endif %}
          width: 25%
          top: 31%
          left: 0%
          border-top-right-radius: 7px
          border-bottom-right-radius: 7px
          border: "2px solid rgba(0, 0, 0, 0.2)"
          padding: 5px
          font-size: 16px
          line-height: 16px
          color: white
          transform: translate(0%,0%)

      - type: state-label
        entity: sensor.dyson_nitrogen_dioxide
        style:
          background-color_template: >-
            {% if states("sensor.dyson_nitrogen_dioxide")|float > 8 %}
            #E53935
            {% elif states("sensor.dyson_nitrogen_dioxide")|float > 6 %}
            #FB8C00
            {% elif states("sensor.dyson_nitrogen_dioxide")|float > 3 %}
            #FFC107
            {% else %}
            #4CAF50
            {% endif %}
          width: 25%
          top: 45%
          left: 0%
          border-top-right-radius: 7px
          border-bottom-right-radius: 7px
          border: "2px solid rgba(0, 0, 0, 0.2)"
          padding: 5px
          font-size: 16px
          line-height: 16px
          color: white
          transform: translate(0%,0%)

      - type: state-label
        entity: sensor.dyson_volatile_organic_compounds
        style:
          background-color_template: >-
            {% if states("sensor.dyson_volatile_organic_compounds")|float > 8 %}
            #E53935
            {% elif states("sensor.dyson_volatile_organic_compounds")|float > 6 %}
            #FB8C00
            {% elif states("sensor.dyson_volatile_organic_compounds")|float > 3 %}
            #FFC107
            {% else %}
            #4CAF50
            {% endif %}
          width: 25%
          border: "2px solid rgba(0, 0, 0, 0.2)"
          top: 59%
          left: 0%
          border-top-right-radius: 7px
          border-bottom-right-radius: 7px
          padding: 5px
          font-size: 16px
          line-height: 16px
          color: white
          transform: translate(0%,0%)

      - type: state-icon
        entity:  sensor.bedroom_humidity_dyson
        style:
          top: 2%
          right: 0px
          background-color: "rgba(0, 0, 0, 0.5)"
          border: "2px solid rgba(0, 0, 0, 0.2)"
          transform: translate(0%,0%)
          width: 20%
          border-top-left-radius: 7px
          border-bottom-left-radius: 7px

      - type: state-label
        entity:  sensor.bedroom_humidity_dyson
        style:
          top: 2%
          right: 5%
          transform: translate(0%,0%)
          line-height: 25px

      # Temperature and Humidity

      - type: state-icon
        entity:  sensor.bedroom_temperature_dyson
        style:
          top: 16%
          right: 0px
          background-color: "rgba(0, 0, 0, 0.5)"
          border: "2px solid rgba(0, 0, 0, 0.2)"
          transform: translate(0%,0%)
          width: 20%
          border-top-left-radius: 7px
          border-bottom-left-radius: 7px

      - type: state-label
        entity:  sensor.bedroom_temperature_dyson
        style:
          top: 16%
          right: 3%
          transform: translate(0%,0%)
          line-height: 25px

      # Filters

      - type: custom:circle-sensor-card
        entity: sensor.dyson_carbon_filter
        name: Carbon Filter
        fill: rgba(0, 0, 0, 0.7)
        style:
          top: 25%
          left: 50%
          width: 20%

      - type: custom:circle-sensor-card
        entity: sensor.dyson_hepa_filter
        name: HEPA Filter
        fill: rgba(0, 0, 0, 0.7)
        style:
          top: 60%
          left: 50%
          width: 20%{% endraw %}
```
