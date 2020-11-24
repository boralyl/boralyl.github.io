---
title: Automating A Light Switch With Old Wiring
excerpt: >
  How to automate a light switch with old wiring and a non-standard light bulb.
categories:
  - home automation
tags:
  - homeassistant
  - home automation
  - lighting
---

One of the things that had been eluding me, in terms of automation, was my front
porch light.  It uses an array of LEDs as it's light source so it's not something
that you can simply swap in a Philips Hue bulb.  My goal was to be able to automatically
turn it on at dusk.

Once I realized I couldn't just put in a smart bulb, I found what I thought would
be a workable solution.  The [Shelly 1](https://www.amazon.com/SHELLY-Wireless-Automation-Android-Application/dp/B07G33LNDY)
is a device that resides behind your light switch.  You wire it up with your
existing wiring and it provides WiFi along the path.  With this you can then
toggle on/off the light connected to the switch.

I didn't know much about wiring, but after a youtube crash course and reading
several articles it seemed do-able.  After I turned off the power at the breaker,
I pulled out the light switch and unscrewed the wires from the switch in order
to get a better look at what I was working with.  To my dismay there was no neutral
wire.  Even after testing the wires with a multimeter I still couldn't figure out
which was ground and which was hot.  Both were giving similar readings.  Since
I'm not an electrician I decided to scrap the idea.

A few weeks later after talking with some co-workers about my problem, one mentioned that he thought
he had seen a switch of some sort that sits on top of your light switch and can
turn it on and off.  I did a little more research to see what other products
were out there that could solve this issue.  That's when I stumbled upon the various
devices that sit over top your switch and have a motor which can manually press
the rocker on or off.  I looked around and settled on the [THIRDREALITY Smart Light Switch Gen 2](https://www.amazon.com/RealitySwitch-Plus-RealityAdapter-included-assistants/dp/B07K3TRG6W).
It operates on a zigbee netwowrk which I already had running in homeassistant
with my Xioami and Ikea devices, so it seemed like a good fit.

[![THIRDREALITY Light Switch Plus](/assets/images/0002_thirdreality.jpg)](/assets/images/0002_thirdreality.jpg)

It works with both rocker switches and your standard US up/down light switch.
It took a minute or so to install and then I went about pairing it with the builtin
ZHA service in homeassistant.  To my surprise it paired easily and showed up in
the zigbee integration panel as `Third Reality, Inc 3RSS008Z`.  I was then able
to easily toggle it via the lovelace UI and add it in to automations in Node-Red.

It's not a perfect device.  It's a little bulky and not exactly quiet.  It has a 
mechanical 'whirring' noise when it toggles the light on/off.  Despite that, I'm
very pleased with how it operates and would wholeheartedly recommend it to anyone else in a similar
situation. 

I will also say that support for the product was fantastic. They
went above and beyond to help me debug why my second switch would not pair.  This
is despite me telling them that I was not using their hub and was using a custom
solution with the [Conbee USB stick](https://www.amazon.com/CONBEE-INTELLIGENT-ZIGBEE-GATEW-Pack/dp/B07MYV4FHW/ref=sr_1_1?keywords=conbee+usb&qid=1561342746&s=hi&sr=1-1) (at the time of writing this is no
longer available, get the [Conbee II](https://shop.dresden-elektronik.de/conbee-2.html?___store=english&___from_store=deutsch) instead).
