---
title: "Use CoordinatorEntity when using the DataUpdateCoordinator"
excerpt: >
  A quick tip on using the CoordinatorEntity class for you entities when using the
  DataUpdateCoordinator in Home Assistant.
categories:
  - home automation
tags:
  - homeassistant
  - development
  - custom_component
header:
  image: /assets/images/0019_header.jpg
  image_description: Network Usage
  caption: Network usage to the searching domain.
---

I recently had a user point out that one of my custom components that monitors a user's
[Nintendo Wish List](https://github.com/custom-components/sensor.nintendo_wishlist) was
using excessive data. He pointed out the drop in traffic to the domain that is used
to search for games on sale in Europe when he disabled the integration as you can see
below and in the header image.

[![Data Usage](/assets/images/0019_header.jpg)](/assets/images/0019_header.jpg)

I was a bit perplexed as the custom integration utilized Home Assistant's
[DataUpdateCoordinator](https://developers.home-assistant.io/docs/integration_fetching_data/#coordinated-single-api-poll-for-data-for-all-entities) which drastically reduces network
calls by fetching all of the data needed by the entities just once. The entities then
use the data stored by the coordinator to update their state. The other way to do this
is to have each entity (think 10 games on your wish list) and each one individually
hits the api to see if it's on sale. Since all the data comes from the same endpoint we
only need to make that call once and the `DataUpdateCoordinator` helps us manage that.

One of the arguments that is passed to the `DataUpdateCoordinator` is a `datetime.timedelta` that specified how
often it should update. I have this value configurable through the integration and
defaulted to once an hour. After doing some debugging I saw thousands of updates were
being made in the first few hours.

The TLDR is if you are using the `DataUpdateCoordinator`, your entities need to make
sure that they return `False` for the `should_poll` property. By default this value is
`True` when you sub-class the `Entity` or `BinarySensorEntity`. Without this, data will
be fetched about every 30 seconds.

While I was investigating this issue I stumbled upon the [CoordinatorEntity](https://github.com/home-assistant/core/blob/dev/homeassistant/helpers/update_coordinator.py#L291)
which was added at some point in the Fall of 2020. This class basically just takes care
of defining a few common methods you would normally add to all of your `Entity` classes
when you are using the `DataUpdateCoordinator`. It also explicitly sets the `should_poll`
property to `False` which was the hint I needed to figure out why my integration was
making so many network calls.

If you are interested in the details, the commit to switch to sub-classing the `CoordinatorEntity` can
be found [here](https://github.com/custom-components/sensor.nintendo_wishlist/commit/6709a5c1b6e323494e7449fa1ac24e61100fc302).
Hopefully this helps someone else out if they run into this issue with their custom
component.
