---
title: Restoring An Entity on Home Assistant Restart
excerpt: >
  How to restore an entity in home assistant on restart in your custom component.
categories:
  - programming
tags:
  - homeassistant
  - python
---

## The Problem
Recently I was developing a custom component which does some web scraping to
get the status of my virtual punchard at my crossfit gym.  It grabs the number
of classes remaining as the state for the sensor then stores some other attributes
like the expiration date, if it's expired, classes remaining and total classes purchased.

It's values really will only change at most from day to day.  Since I don't need
to update the state and attributes of the sensor often (and it would be a waste
to do so), I set the `SCAN_INTERVAL` to do updates once a day.  The sensor would
get the correct values and then I could use them to create a simple card like below.

[![Simple HA Lovelace Card](/assets/images/0005_crossfit_card.png)](/assets/images/0005_crossfit_card.png)

The problem I ran into is when I restarted Home Assistant for upgrades or config
changes, all of the values for the sensor would revert to empty values.  On top of that, without
doing a force update through the dev services panel I would have to wait a full
day for the values to get populated again. 

## RestoreEntity
Home Assistant, in general, has excellent documentation.  However, I could not
find any answer for this particular problem when searching the [developer docs](https://developers.home-assistant.io/doc)
or the [community forums](https://community.home-assistant.io/).  After browsing
the code I found a helper class that looked like it would address the exact problem
I was facing.  It is named [RestoreEntity](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/restore_state.py#L217)
and can be used as a mixin on your `Entity` class.  I found a few examples of
this class being used within the Home Assistant codebase, which showed me that
in addition to using the mixin class we also needed to override the 
[asnyc_added_to_hass](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/entity.py#L406) method.

I implemented the method which you can see below.  For brevity I've only included 
the important bits of my entity class. 

```python
class CrossfitSensor(RestoreEntity):
    """Representation of a sensor."""

    def __init__(self, hass, username, password):
        self._hass = hass
        self.username = username
        self.password = password
        self.attrs = {}
        self._state = None

    @property
    def name(self):
        """Return the name of the sensor."""
        return 'Crossfit Classes'

    @property
    def unit_of_measurement(self):
        """Return the unit of measurement of this entity, if any."""
        return 'classes remaining'

    @property
    def state(self):
        """Return the state of the sensor."""
        if self._state is None:
            if self.attrs:
                if self.attrs['expired']:
                    self._state = 0
                else:
                    self._state = self.attrs['remaining']
        return self._state

    @property
    def device_state_attributes(self):
        return self.attrs

    async def async_added_to_hass(self) -> None:
        """Handle entity which will be added."""
        await super().async_added_to_hass()
        state = await self.async_get_last_state()
        if not state:
            return
        self._state = state.state

        async_dispatcher_connect(
            self._hass, DATA_UPDATED, self._schedule_immediate_update
        )

    @callback
    def _schedule_immediate_update(self):
        self.async_schedule_update_ha_state(True)
    
    def update(self):
        """Get the latest data and updates the state."""
        # omitted for brevity
```

I needed to add the `async_added_to_hass` and `_schedule_immediate_update` methods.
`async_added_to_hass` gets called when an entity is added to a platform.  So the
above code gets the previous state by calling `async_get_last_state` (this method
is provided by the `RestoreEntity` mixin) then sets the `_state` attribute.  Finally
it issues an event call to let Home Assistant know that data for the entity has
been updated.

I restarted Home Assistant expecting to see the sensor state and attributes populated,
but instead they were again reverted to empty values.

## The Solution

So what did I do wrong?  It turns out that the actual code in the `async_added_to_hass`
method was working just fine.  It was appropriately setting the state of the entity
to `0`, since the punchcard was expired.  I had assumed that the `RestoreEntity`
would magically also populate my device state attributes stored in `self.attrs`.

Before calling `async_dispatcher_connect` I needed to also set the attributes.
It turns out this data is stored in the state instance's `attributes` property.
I modified my method to set those like below.

```python
async def async_added_to_hass(self) -> None:
    """Handle entity which will be added."""
    await super().async_added_to_hass()
    state = await self.async_get_last_state()
    if not state:
        return
    self._state = state.state

    # ADDED CODE HERE
    if 'expired' in state.attributes:
        self.attrs = {
            'expiration_date': datetime.strptime(
                state.attributes['expiration_date'], '%Y-%m-%dT%H:%M:%S'),
            'expired': state.attributes['expired'],
            'purchased': state.attributes['purchased'],
            'remaining': state.attributes['remaining'],
        }

    async_dispatcher_connect(
        self._hass, DATA_UPDATED, self._schedule_immediate_update
    )
```

I had to explicitly check that one of the known keys was already in the state's
attributes before attempting to access them.  If an update of values for the
sensor had not yet run this would not have these keys in them.  Then I simply
set the dict on my `self.attrs` attribute.  I then restarted Home Assistant and my
values were still present in the dashboard.

## TLDR

To retain state and attributes between restarts for your entity:

* Make your entity class subclass the [`RestoreEntity`](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/restore_state.py#L217) mixin class.
* Override the [`asnyc_added_to_hass`](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/entity.py#L406) method. This method should call the [`RestoreEntity.async_get_last_state`](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/restore_state.py#L236) method of the mixin class to retrieve the last state.  Take this instance and set your state and attribute values on your entity class.
* Finally let home assistant know you've updated data by calling [`async_dispatcher_connect`](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/dispatcher.py#L32) with a callback function that forces an update using [`async_schedule_update_ha_state`](https://github.com/home-assistant/home-assistant/blob/dev/homeassistant/helpers/entity.py#L342).
