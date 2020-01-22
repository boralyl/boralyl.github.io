---
title: "Integrating the Roborock S4 in Home Assistant"
excerpt: >
  The issues I ran into and how I was able to integrate the Roborock S4 vacuum
  cleaner in Home Assistant.
categories:
  - home automation
tags:
  - vacuum
  - homeassistant
  - home automation
  - roborock
header:
  image: /assets/images/0007_header.png
  image_description: Vacuum Lovelace UI
  caption: "Roborock S4 Vacuum (credit: https://en.roborock.com/)"
classes: wide
---
*Note: this post was written about the Roborock S4 but it should also work with the S5, S5 Max, S6 and S6 Pure.*
{: .notice--warning}

<div><p>This is part 1 of a 3 part post on the vacuum.  I"ll update the links below as they are
posted.</p>
<ul>
  <li>Part 1 - Integrating the Roborock S4 in Home Assistant (reading now!)</li>
  <li>Part 2 - Vacuum Automations and Scripts (coming soon)</li>
  <li>Part 3 - Vacuum Lovelace Dashboard (coming soon)</li>
</ul>
</div>
{: .notice--info}

I recently acquired a [Roborock S4](https://en.roborock.com/pages/roborock-s4) as a gift.
It has a fantastic integration in Home Assistant, so I was excited to get started.
It wasn't as straight forward as the steps listed in the documentation. The
following post describes the steps I took to get it integrated.

The [documentation](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio) for configuring the vacuum is pretty detailed and provides
several options for the hardest part, which is [retrieving the access token](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#retrieving-the-access-token)
needed to send commands to the vacuum.

I was able to side-load the required version
of the android Mi Home app without any issue.  I was also able to find the access
token in the logs on the sd-card.  The problem I ran into was that I wasn't actually
able to add the vacuum in the Mi Home app.  On the final step within the app I would always
recieve the following error: `Failed to start extension, try again`.  Technically
I could have just used the access token and started automating the vacuum, but
I would have not been able to use any of the map features like creating zones to
be cleaned.

Using the roborock app reccomended by the manufacturer I was able to successfully
add the vacuum, but there was no way to retrieve the access token.  In order to
add the vacuum to either app you have to reset it's wifi settings which causes the
previous access token to be revoked.

### How I Obtained the Access Token and Generated the Map

After a bit of research I came across a modified version of the Mi Home app that
not only made adding the vacuum work, but also made it dead simple to obtain the
access token and see what commands are sent to the vacuum from the app.  This
was useful as I was able to figure out how to do certain actions not provided by
the Home Assistant integration, like resetting the hours for when parts need to
be replaced. It also made getting the parameters for a zone super simple.
 The [website](https://www.kapiba.ru/2017/11/mi-home.html) is in
Russian but can be [easily translated to English](https://translate.google.com/translate?hl=en&sl=auto&tl=en&u=http%3A%2F%2Fwww.kapiba.ru%2F2017%2F11%2Fmi-home.html%3Ffbclid%3DIwAR2j_xmuskJSphPDWDosfa4z7R3fZnvoaxYa8PxtqCgGj7HSBx0DUkg7hyw).
The modified APK can be found on the site under the `DOWNLOAD VERSION 5.X.X..*`
link.  The password to download the file can be found at the bottom of the page.
As of writing, the author has posted an updated version on 2020-01-03.

The first thing I did was create the desired directory path mentioned in the
translated post (*Note: this is no longer in the post, so it may not be required anymore*).  To do this I used [adb](https://developer.android.com/studio/command-line/adb), but this could also be down with a
file browser app.

```sh
$ adb shell
htc_pmewl:/ $ cd /sdcard/
htc_pmwell:/sdcard $ mkdir -p vevs/logs
```

I then side-loaded the APK on to a spare phone, started it up, and followed
the normal process to add a new vacuum.  I was able to add it without any issue.
Afterwards I tailed the created log file to obtain the access token.

```sh
$ adb shell
htc_pmewl:/ $ cd /sdcard/vevs/logs
htc_pmewl:/sdcard/vevs/logs $ tail -f 261880538.txt
192.168.1.111
roborock.vacuum.s4
d41d8cd98f00b204e4400998ecf8427e

2019-12-26 18:12:12 -> {"id":6156,"method":"enable_log_upload","params":[0,2]}
2019-12-26 18:12:12 <- {"code":0,"message":"ok","result":["ok"],"id":6156}
2019-12-26 18:12:15 -> {"id":6157,"method":"app_get_init_status","params":[]}
...
```
You can see the IP address assigned to the vacuum, the model id and most importantly
the access token (*the values above have been altered from what's on my actual device*).

Now that I had the access token the last step was to update my `configuration.yaml`,
[per the documentation](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#configuration), and restart home assistant.

```yaml
vacuum:
  - platform: xiaomi_miio
    host: 192.168.1.111
    token: d41d8cd98f00b204e4400998ecf8427e
    name: Roborock S4
```

Next I used the modified version of the app and I ran the vacuum once in order
for it to use the lidar to map out the floor.  With this I was ready to create
the zones for each room.

[![Vacuum Map](/assets/images/0007_vacuum_map.png)](/assets/images/0007_vacuum_map.png)

### Obtaining Parameters for Zones

The modified version of the Mi Home app will output all of the commands done in
the app in the log file.  After I figured out what the command was by creating
zones in the app and monitoring the log file, I came up with a grep command to
make it very simple to extract the parameters for each zone I defined in the app.
This uses adb, but you could also use a file browser and search for `app_zoned_clean`
to find the parameters.

```sh
$ adb shell
htc_pmewl:/ $ cd /sdcard/vevs/logs
htc_pmewl:/sdcard/vevs/logs $ tail -f 261880538.txt | grep app_zoned_clean
2019-12-29 13:35:55 -> {"id":5451,"method":"app_zoned_clean","params":[[22050,24850,29800,29150,1]]}
```

So while I tailed the log file, in the app I defined a zone on my map then tapped
on the `Clean` button.  After doing that I got the output you can see above which
contains the parameters I need to send to the vacuum to clean a zone using the
[xiaomi_miio.vacuum_clean_zone](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#service-xiaomi_miiovacuum_clean_zone)
service. Note that in Home Assistant you only need to provide the first 4 numbers
for that service call.  The last number you see is the number of times to clean
the zone.  For reference this is what those 5 numbers map to: `[x1, y1, x2, y2, repetitions]`.
The `x` and `y` coordinates define the box you created in the app. I noted the
numbers for each zone I created and repeated the process until I had
the parameters for all of the rooms on the map.

**One thing that is very important** is that after you obtain these parameters you
should not do a full clean again either through the Mi Home app, Home Assistant's
more details modal, or by pressing the clean button on top of the vacuum.  A full
clean will recreate the map and could modify the coordinates.  This would force
you to recreate the zones to get the parameters again.  Instead, to initiate a full clean the [xiaomi_miio.vacuum_clean_zone](https://www.home-assistant.io/integrations/vacuum.xiaomi_miio/#service-xiaomi_miiovacuum_clean_zone)
service should be used and you should pass it an array of each zone's parameters.

In the 2nd post I will share the automations and scripts I've created to
use with the Roborock S4 vacuum.  In the final post I'll share my lovelace configuration
for the cards I used to create my vacuum dashboard.
