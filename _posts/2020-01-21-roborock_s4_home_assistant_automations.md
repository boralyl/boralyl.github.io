---
title: "Roborock S4 Automations and Scripts in Home Assistant"
excerpt: >
  This post documents the various automations and scripts I've created for
  interacting with my Roborock S4 vacuum in Home Assistant.
categories:
  - home automation
tags:
  - vacuum
  - homeassistant
  - home automation
  - roborock
  - lovelace
header:
  image: /assets/images/0008_header.png
  image_description: Node-RED Flow
  caption: "Node-RED flow"
toc: true
---
*Note: this post was written about the Roborock S4 but it should also work with the S5, S5 Max, S6 and S6 Pure.*
{: .notice--warning}

<div><p>This is part 2 of a 3 part post on the vacuum.  I"ll update the links below as they are
posted.</p>
<ul>
  <li><a href="https://aarongodfrey.dev/home%20automation/integrating_the_roborock_s4_in_home_assistant/">Part 1 - Integrating the Roborock S4 in Home Assistant</a></li>
  <li>Part 2 - Roborock S4 Automations and Scripts in Home Assistant (Reading Now!)</li>
  <li><a href="https://aarongodfrey.dev/home%20automation/roborock_s4_lovelace_dashboard/">Part 3 - Roborock S4 Lovelace Dashboard</a></li>
</ul>
</div>
{: .notice--info}

In this post I'm documenting the various automations and scripts I've created to
use with my [Roborock S4 Vacuum](https://en.roborock.com/pages/roborock-s4).  I
had a general idea of some of the automations I wanted to do right away.  After
browsing the [Home Assistant Community Forums](https://community.home-assistant.io/)
I came up with several others.  Hopefully these automations give you a taste of
what can be done and inspire you to create your own automations.

## Automations

The automations that I'm documenting here are written using [Node-RED](https://flows.nodered.org/node/node-red-contrib-home-assistant-websocket), but
they should also be able to be written in the Home Assistant yaml format as well.
A JSON export of each flow will be included with each automation so you can
import them and try them out yourself.  Obviously you will need to manually
make some adjustments to match your particular entities, but it's a good starting
point to get an idea of how they work.

### Auto Running the Vacuum

[![Run Vacuum Automation](/assets/images/0008_header.png)](/assets/images/0008_header.png)

This flow is responsible for auto-running the vacuum.  This is superior to a
schedule as it uses many other variables (besides a day/time) to determine when
to run and if it should even run at all.  This automation gets triggered in one of 3 ways:

1. The flow is triggered manually via the inject node.
2. Everyone has left the house and the vacuum hasn't run yet.
3. It's 2pm, the vacuum hasn't run yet and someone is still home.

The gist of this flow is that the vacuum will only ever get triggered to run once
a day.  If everyone in the house leaves and it hasn't run yet, it will start cleaning.
If 2pm has come around and we haven't left the house yet, it will go ahead and
commence with the cleaning.

I also have a few short circuits in there to prevent the cleaning from starting.
If the vacation mode input boolean is on we don't want the vacuum to run at all.
Also if the vacuum is already running there is no reason to try to restart it.

Finally, one of the limitations of the vacuum, as of writing, is that it can only accept
5 zones to be cleaned with the [xiaomi_miio.vacuum_clean_zone service](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#service-xiaomi_miiovacuum_clean_zone).
I have 6 zones (or rooms), so I set a global variable that stores
that the 6th zone (the guest bedroom) has not yet been vacuumed.  When the vacuum
finishes it's zoned cleaning of the first 5 zones, it will check that global variable and trigger another
zoned cleaning, but with just the remaining room.  There are other ways to do this in Node-RED
like using a [simple message queue](https://flows.nodered.org/node/node-red-contrib-simple-message-queue), but this method is working for me at the moment.

[JSON Export of Flow](/assets/data/auto_run_vacuum.json)

### Toggling Roborock Daily Run Boolean

[![Toggling Roborock Daily Run Boolean](/assets/images/0008_toggling_daily_run_boolean.png)](/assets/images/0008_toggling_daily_run_boolean.png)

The name of these automations is a little misleading, this actually does 4 things.

1. When the vacuum finishes cleaning, it checks the global variable to see if it's
   cleaned the last zone yet.  If it has, it sets an input boolean to `true` to
   indicate the vacuum has run for the day.  This prevent any other runs that
   could be triggered by time or by the house being unoccupied.
2. When the vacuum finishes cleaning, it checks the global variable to see if it's
   cleaned the last zone yet.  If so it will set another input boolean
   to `false`.  This is for the empty dustbin indicator,  you can see how that
   works [below](#empty-dustbin-notification).
3. When the vacuum finishes cleaning, it checks the global variable to see if it's
   cleaned the last zone yet.  If it hasn't, it triggers another zoned cleaning with
   the final zone/room.  Then it sets the global variable to `true` so that when
   the vacuum finishes again the automation will run the 2 steps above and the
   vacuum is done for the day.
4. The flow on the bottom runs every day at midnight.  This resets the daily run
   input boolean so that the vacuum is ready to run again for the next day.

[JSON Export of Flow](/assets/data/toggling_roborock_daily_run_boolean.json)

### Empty Dustbin Notification

[![Maintenance Notifications](/assets/images/0008_dustbin_notification.png)](/assets/images/0008_dustbin_notification.png)

Unfortunately the vacuum doesn't expose any way to know if the dustbin is full
or if it has been emptied recently.  However, based on my automations I know when
the vacuum runs.  So when it finishes cleaning all the rooms, it will check if
anyone is home or not.  If no one is home, it sets an input boolean to `false`
indicating that the dustbin has not been emptied.

Then whenever someone arrives home it will kick off a 10 minute delay (to allow the
individual to get in the door and unpack) and then check if that input boolean
for dustbin emptied is `false` or not.  If it's `false`, it will send a push notification
with a message reminding the individual to empty the vacuum dustbin.  You could
easily modify this to announce the message via a smart speaker.

[JSON Export of Flow](/assets/data/dustbin_notification.json)

### Maintenance Notifications

[![Maintenance Notifications](/assets/images/0007_automation_maintenance_notifications.png)](/assets/images/0007_automation_maintenance_notifications.png)

This flow handles sending notifications when vacuum parts need to be replaced or
cleaned.  Anytime the vacuum's state is updated 4 checks are done to see if the
number of hours remaining for each part or sensor is <= 10 hours.  If so, it will
send a push notification with details on which part and how soon it will need to
be replaced or cleaned.

[JSON Export of Flow](/assets/data/maintenance_notifications.json)

### Update Last Run Sensor

[![Maintenance Notifications](/assets/images/0007_automation_update_last_run_sensor.png)](/assets/images/0007_automation_update_last_run_sensor.png)

This is a pretty simple flow that triggers every 5 minutes in order to update
the value for the custom template sensor `sensor.roborock_s4_last_cleaned`.
This sensor value is used in the [lovelace vacuum card](/assets/images/0007_lovelace_vacuum_card.png)
that I'll discuss in the next post.  The sensor is defined as the following:

```yaml
- platform: template
  sensors:
    # Displayed on the vacuum card
    # NOTE: This date is converted to be timezone aware so that it plays nice
    # with some other templating functions and filters.
    roborock_s4_last_cleaned:
      friendly_name: Relative time since last cleaning ended
      value_template: {% raw %}"{{relative_time(strptime(as_timestamp(state_attr('vacuum.roborock_s4', 'clean_stop'))|timestamp_custom('%Y-%m-%d %H:%M:%S%z'), '%Y-%m-%d %H:%M:%S%z'))}}"{% endraw %}
```

The reason we need this is because typically the vacuum's state does not change
after the vacuum has run for the day.  This sensors displays a value with relative
time like `5 hours ago`.  So we need to manually trigger a sensor update to keep
that value constantly updated.

[JSON Export of Flow](/assets/data/update_sensor_value.json)

## Scripts

The scripts that I'm documenting here are written using [Node-RED](https://flows.nodered.org/node/node-red-contrib-home-assistant-websocket), but
they should also be able to be written in the Home Assistant yaml format as well.

Both of these scripts are triggered via MQTT.  There are other ways to do this,
but I already had MQTT setup on my Home Assistant instance so this was the most
straightforward way to trigger these scripts with a JSON payload.

### Vacuum Room

[![Vacuum Room](/assets/images/0007_script_vacuum_room.png)](/assets/images/0007_script_vacuum_room.png)

This script accepts a JSON payload that contains the name of a room and an integer
representing the number of times to clean the room.  It uses a function node
to create the payload for the [xiaomi_miio.vacuum_clean_zone](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#service-xiaomi_miiovacuum_clean_zone)
service call.  This script is triggered via the [lovelace vacuum room card](/assets/images/0007_lovelace_vacuum_room_card.png) that will be discussed in the next post.

[JSON Export of Flow](/assets/data/vacuum_room.json)

### Vacuum First Floor

[![Vacuum Room](/assets/images/0007_script_vacuum_first_floor.png)](/assets/images/0007_script_vacuum_first_floor.png)

This script will trigger a full cleaning of the first floor.  As I mentioned
previously in [part 1](https://aarongodfrey.dev/home%20automation/integrating_the_roborock_s4_in_home_assistant/#obtaining-parameters-for-zones), if you initiate a full clean via Home Assistant, the Mi Home app, or
by pressing the button on top of the vacuum, it will redraw the map.  If you had
setup zones, this could possibly mean that you would need to set them up again
as the coordinates may have slightly changed.

This script simply does a zoned cleaning of the 5 predefined zones I've created.
Since the zoned cleaning can only accept a max of 5 zones, it also sets the global
variable to vacuum the guest bedroom to `false`.  This will cause it to vacuum
that room after it finishes the other 5 zones.

This script gets triggered via an icon button in my [lovelace vacuum card](/assets/images/0007_lovelace_vacuum_card.png)
which I will discuss in the next post.

[JSON Export of Flow](/assets/data/vacuum_first_floor.json)
