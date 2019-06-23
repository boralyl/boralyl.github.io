---
title: "My Home Assistant Floorplan"
excerpt: >
  Walking through how my home assistant floorplan came to fruition and the
  cool things that it does.
categories:
  - home automation
tags:
  - floorplan
  - homeassistant
  - home automation
header:
  image: /assets/images/0001_header.png
  image_description: 3D Rendered Floorplan
  caption: 3D Rendered Floorplan.
---

During the past 6 months I've really gained an interest in home automation.
I discovered [home assistant](https://www.home-assistant.io/) and have a nearly
endless list of projects to work on.  While browsing the [/r/homeassistant](https://www.reddit.com/r/homeassistant/)
subreddit I discovered the ability to utilize the new lovelace UI to create an
interactive 2D floorplan.  You can view a basic example in the excellent
[lovelace documentation](https://www.home-assistant.io/lovelace/picture-elements/).

[![My 2D Floorplan](/assets/images/0001_2dfloorplan.png)](/assets/images/0001_2dfloorplan.png)

The 2D floorplan worked great.  I could have a room lighten up when the lights
were on, as well as toggling them on and off.  I was also able to see sensors values like
temperature and motion sensor events.  The floorplan works well for giving you
a quick overview of the lights/sensors in your house.

I was pleased with what I had created, but then I saw a few posts on reddit where 
users had created a 3D floorplan with rendered images that also included realistic lighting.  
[This post](https://www.reddit.com/r/homeassistant/comments/bommbe/finally_got_my_floorplan_working_properly_with/)
and [this one,](https://www.reddit.com/r/homeassistant/comments/bqnl1v/floorplan_thank_you_for_inspiring_me/)
in particular, inspired me to create my own 3D floorplan.

After several weeks of experimenting with [SweetHome3D](http://www.sweethome3d.com/)
to create the 3D model, I ended up with a pretty accurate representation of my house.
I then added the floorplan as a [picture-elements card](https://www.home-assistant.io/lovelace/picture-elements/)
and added all of my entities to it.  You can see it in action below.
<iframe width="640" height="360" src="https://www.youtube-nocookie.com/embed/ebMQwVjVewU?controls=0&amp;showinfo=0" frameborder="0" allowfullscreen></iframe>

Some features shown in the video:

* Back door opens in the floorplan when the back door contact sensor reports that it is open and closes when the sensor reports closed.
* Ability to toggle lights and view which lights are on with accurate light rendering.
* Ability to view status of cameras and arm/disarm them.
* Ability to turn the TV(s) on or off as well as showing a glow when they are on.
* Button that toggles between the 2 floors of the house.
* Displays sensor values for temperature and motion sensor events.

Because of the angle I used to display my floorplan I was not able to add a
button on the floorplan itself to toggle the kitchen counter lights.  To get
around this, I created several separate renders of the kitchen counter and made
that into an element in the `picture-elements` card. The image accurately updates
based on the state of the kitchen and counter lights.

I plan on writing two future posts that will cover how you can create your own
3D floorplan for use in home assistant.  The first post will cover some tips
and tricks for using SweetHome3D to create your house and render the images needed.
In the second post I will share my lovelace configuration and explain how I
implemented the various features in my floorplan.
