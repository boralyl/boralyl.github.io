---
title: "Building a Home Assistant Custom Component Part 4: Options Flow"
excerpt: >
  Part 4 of building a custom component in Home Assistant.  In this post we'll
  examine how to add an options flow so that your component can have additional options
  configured through the configuration UI in Home Assistant.
categories:
  - home automation
tags:
  - homeassistant
  - development
  - tutorial
  - custom_component
  - config flow
  - options flow
header:
  image: /assets/images/0015_header.png
  image_description: "Options flow UI for GitHub Custom component"
  caption: "Options flow UI for GitHub Custom component."
toc: true
---

<div>
  <p>This is the fourth part of a multi-part tutorial to create a Home Assistant custom component.</p>
  <ul>
    <li><a href="https://aarongodfrey.dev/home%20automation/building_a_home_assistant_custom_component_part_1/">Part 1 - Project Structure and Basics</a></li>
    <li><a href="https://aarongodfrey.dev/home%20automation/building_a_home_assistant_custom_component_part_2/">Part 2 - Unit Testing and Continuous Integration</a></li>
    <li><a href="https://aarongodfrey.dev/home%20automation/building_a_home_assistant_custom_component_part_3/">Part 3 - Config Flow</a></li>
    <li>Part 4 - Options Flow (Reading Now!)</li>
  </ul>
</div>
{: .notice--info}

## Introduction

In this post we will be adding an [Options flow](https://developers.home-assistant.io/docs/config_entries_options_flow_handler/)
to our custom component. We are still using the same example project, [github-custom-component](https://github.com/boralyl/github-custom-component-tutorial).
You can find the diff for this post on the [feature/part4](https://github.com/boralyl/github-custom-component-tutorial/compare/feature/part3...feature/part4) branch.

The options flow allows a user to configure additional options for the component at any
time by navigating to the integrations page and clicking the `Options` button on the
card for your component. Generally speaking these configuration values are optional, whereas
values in the config flow are required to make the component function.

I highly suggest reading over the [official documentation](https://developers.home-assistant.io/docs/config_entries_options_flow_handler/)
prior to continuing along with the tutorial.

## Enable Options Support

Per the [documentation](https://developers.home-assistant.io/docs/config_entries_options_flow_handler/#options-support)
the first step is to define a method on your config flow class that let's it know that the
component supports options. In our case we will add this to our [GithubCustomConfigFlow](https://github.com/boralyl/github-custom-component-tutorial/blob/master/custom_components/github_custom/config_flow.py#L120-L124) class.

```python
@staticmethod
@callback
def async_get_options_flow(config_entry):
    """Get the options flow for this handler."""
    return OptionsFlowHandler(config_entry)
```

One slight modification from the official documentation is that our `OptionsFlowHandler`
class will require the instance of the config entry when initializing. This will be required
for nearly every component you may write as we will use the `options` property of the
`config_entry` to populate default values for our options flow form.

## Configure Fields and Errors in strings.json

Just like our config flow, we need to name our data fields and error messages in the
`strings.json`. These will be nested under an `options` key. You will need to add these
for each language you choose to support in the `translations` directory.

```diff
+  },
+  "options": {
+    "error": {
+      "invalid_path": "The path provided is not valid. Should be in the format `user/repo-name` and should be a valid github repository."
+    },
+    "step": {
+      "init": {
+        "title": "Manage Repos",
+        "data": {
+          "repos": "Existing Repos: Uncheck any repos you want to remove.",
+          "path": "New Repo: Path to the repository e.g. home-assistant-core",
+          "name": "New Repo: Name of the sensor."
+        },
+        "description": "Remove existing repos or add a new repo."
+      }
+    }
```

## Define an OptionsFlow Handler

The next step is to write our class to handle the options flow. This will look very similar
to the class we defined for our config flow so it should be familiar. For brevity I'm
going to omit much of the logic in the class to try to simplify it to show the important
parts. I'll give a general overview of how it works then I'll dive into the specific
logic I [added for our tutorial component](#options-flow-in-the-github-custom-component).

```python
class OptionsFlowHandler(config_entries.OptionsFlow):
    """Handles options flow for the component."""

    def __init__(self, config_entry: config_entries.ConfigEntry) -> None:
        self.config_entry = config_entry

    async def async_step_init(
        self, user_input: Dict[str, Any] = None
    ) -> Dict[str, Any]:
        """Manage the options for the custom component."""
        errors: Dict[str, str] = {}
        # Grab all configured repos from the entity registry so we can populate the
        # multi-select dropdown that will allow a user to remove a repo.
        entity_registry = await async_get_registry(self.hass)
        entries = async_entries_for_config_entry(
            entity_registry, self.config_entry.entry_id
        )
        # Default value for our multi-select.
        all_repos = {e.entity_id: e.original_name for e in entries}
        repo_map = {e.entity_id: e for e in entries}

        if user_input is not None:
            # Validation and additional processing logic omitted for brevity.
            # ...
            if not errors:
                # Value of data will be set on the options property of our config_entry
                # instance.
                return self.async_create_entry(
                    title="",
                    data={CONF_REPOS: updated_repos},
                )

        options_schema = vol.Schema(
            {
                vol.Optional("repos", default=list(all_repos.keys())): cv.multi_select(
                    all_repos
                ),
                vol.Optional(CONF_PATH): cv.string,
                vol.Optional(CONF_NAME): cv.string,
            }
        )
        return self.async_show_form(
            step_id="init", data_schema=options_schema, errors=errors
        )
```

### Override \_\_init\_\_

We must override `__init__` so that it can accept a `config_entry` instance which we
set as an attribute on the class. As mentioned above this is so we can access it's
`options` property to pre-populate data in our options flow form.

```python
def __init__(self, config_entry: config_entries.ConfigEntry) -> None:
    self.config_entry = config_entry
```

### Define the Options Data Schema

Next up we define our options data schema. This is identical to how we define the schema
for our config flow. We are defining the schema in the method itself so that we can supply
a default value for the repos key which is dynamically evalulated in this method. If you
do not need any dynamic values you can define it as a constant above just like we did with
the schema for the config flow.

```python
options_schema = vol.Schema(
    {
        vol.Optional("repos", default=list(all_repos.keys())): cv.multi_select(
            all_repos
        ),
        vol.Optional(CONF_PATH): cv.string,
        vol.Optional(CONF_NAME): cv.string,
    }
)
```

While I am not using default values for the other keys in the schema in my component, this is where you
would generally look up existing options values from the config entry instance to set
default values for your form. For example:

```python
vol.Optional(CONF_PATH, default=self.config_entry.options[CONF_PATH])
```

### Display the Options Form

There isn't anything new here that we haven't seen in the config flow. One thing to note
that is different from the config flow, is that the options flow only ever has a single
step named `init`.

```python
return self.async_show_form(
    step_id="init", data_schema=options_schema, errors=errors
)
```

### Save Options Data

When a user has submitted `user_input` that validates we can then format and save our
options data by returning the `asnyc_create_entry` method.

```python
# Value of data will be set on the options property of our config_entry
# instance.
return self.async_create_entry(
    title="",
    data={some_option: user_input["some_option"]},
)
```

## Register Options Update Listener

In order for our component to know that options have changed and to be able to act on them,
we must register and update listener when initially setting up our config entry. In our
`__init__.py` file we will define our update listener function and register it with the
config entry.

```python
async def options_update_listener(
    hass: core.HomeAssistant, config_entry: config_entries.ConfigEntry
):
    """Handle options update."""
    await hass.config_entries.async_reload(config_entry.entry_id)
```

As you can see above the logic of the listener is very simple. It just reloads the config
entry so that it can act on the new options data that was saved. We must then register
the listener in our `async_setup_entry` function.

```python
hass_data = dict(entry.data)
# Registers update listener to update config entry when options are updated.
unsub_options_update_listener = entry.add_update_listener(options_update_listener)
# Store a reference to the unsubscribe function to cleanup if an entry is unloaded.
hass_data["unsub_options_update_listener"] = unsub_options_update_listener
hass.data[DOMAIN][entry.entry_id] = hass_data
```

The `add_updated_listener` method returns an unsubscribe function that we will store for
later so that we can clean up the listener if the config entry is removed by the user.

One thing to note is that the update listener function will only get called if the data
passed to `self.async_create_entry` in our Options Flow class is different then it
previously was. If nothing changed, the options update listener will not get called and
your config entry will not be reloaded.

## Use Options Values During Setup

Now that we've setup our options flow, the user can enter values and they will be saved
on the config entry instance. The last step is to use those values while setting up our
platforms. In our `sensor.py` we could then use the options values to change how our
sensors get setup. An example might look something like the following:

```python
async def async_setup_entry(
    hass: core.HomeAssistant,
    config_entry: config_entries.ConfigEntry,
    async_add_entities,
):
    """Setup sensors from a config entry created in the integrations UI."""
    config = hass.data[DOMAIN][config_entry.entry_id]
    some_option = config_entry.options.get("some_option")
    session = async_get_clientsession(hass)
    github = GitHubAPI(session, "requester", oauth_token=config[CONF_ACCESS_TOKEN])
    sensors = [GitHubRepoSensor(github, repo, some_option) for repo in config[CONF_REPOS]]
    async_add_entities(sensors, update_before_add=True)
```

## Options Flow in the Github Custom Component

Now that I went over the general information on using an options flow I wanted to return
to the custom component we've been building in this tutorial. The options flow I added
performs actions that I haven't seen in any other options flows. Mainly it allows for
removing repos that have been added as well as adding new repos via the options flow form.
Below you can see a screenshot of what it looks like.

[![Options Flow](/assets/images/0015_options_flow.png)](/assets/images/0015_options_flow.png)

The multi-select allows a user to uncheck repos that they want to remove. The other two
inputs allow a user to add a new repo and give it an optional name. Clicking `SUBMIT`
will remove un-checked repos and add any new repo if one was specified.

### Removing a Repo

The logic for removing repos looks like the following in our options flow class:

```python
updated_repos = deepcopy(self.config_entry.data[CONF_REPOS])

# Remove any unchecked repos.
removed_entities = [
    entity_id
    for entity_id in repo_map.keys()
    if entity_id not in user_input["repos"]
]
for entity_id in removed_entities:
    # Unregister from HA
    entity_registry.async_remove(entity_id)
    # Remove from our configured repos.
    entry = repo_map[entity_id]
    entry_path = entry.unique_id
    updated_repos = [e for e in updated_repos if e["path"] != entry_path]
```

We first determine which repos were unchecked by comparing the selected repos to the repos
that were originally configured in our `config_entry`. Then we iterate through each `entity_id`
and remove it from the entity registry first, then from our list of repos initially
configured.

### Adding a Repo

If the user enters a value for the `path` input we will then add a new repo. That logic
is shown below:

```python
if user_input.get(CONF_PATH):
    # Validate the path.
    access_token = self.hass.data[DOMAIN][self.config_entry.entry_id][
        CONF_ACCESS_TOKEN
    ]
    try:
        await validate_path(user_input[CONF_PATH], access_token, self.hass)
    except ValueError:
        errors["base"] = "invalid_path"

    if not errors:
        # Add the new repo.
        updated_repos.append(
            {
                "path": user_input[CONF_PATH],
                "name": user_input.get(CONF_NAME, user_input[CONF_PATH]),
            }
        )
```

If a value was provided we first validate it to ensure it's a real GitHub repo. If it is
not we populate the `errors` dict with the error key defined in our `strings.json`. If
there are no errors we simply append the new repository to the existing list.

### Updating the Sensors

When we succuessfully return from our options flow handler it will pass the list of
updated repos as the data keyword argument. This `dict` will get set in the `options`
property of our `config_entry` instance.

```python
return self.async_create_entry(
    title="",
    data={CONF_REPOS: updated_repos},
)
```

We will access that data when setting up our sensors in `sensor.py`. Before creating
our sensors we augment the initial configuration data with the updated repos which may
have had repos removed or added since the initial configuration in our config flow.

```diff
diff --git a/custom_components/github_custom/sensor.py b/custom_components/github_custom/sensor.py
index 9a62f8a..c893fa2 100644
--- a/custom_components/github_custom/sensor.py
+++ b/custom_components/github_custom/sensor.py
@@ -70,6 +70,9 @@ async def async_setup_entry(
 ):
     """Setup sensors from a config entry created in the integrations UI."""
     config = hass.data[DOMAIN][config_entry.entry_id]
+    # Update our config to include new repos and remove those that have been removed.
+    if config_entry.options:
+        config.update(config_entry.options)
     session = async_get_clientsession(hass)
     github = GitHubAPI(session, "requester", oauth_token=config[CONF_ACCESS_TOKEN])
     sensors = [GitHubRepoSensor(github, repo) for repo in config[CONF_REPOS]]
```

## Unit Tests

Unit testing the options flow isn't terribly different than testing the config flow,
but it does require a few extra steps. The test below tests the case where the user
unchecks an existing repo from the config entry.

```python
@patch("custom_components.github_custom.sensor.GitHubAPI")
async def test_options_flow_remove_repo(m_github, hass):
    """Test config flow options."""
    m_instance = AsyncMock()
    m_instance.getitem = AsyncMock()
    m_github.return_value = m_instance

    config_entry = MockConfigEntry(
        domain=DOMAIN,
        unique_id="kodi_recently_added_media",
        data={
            CONF_ACCESS_TOKEN: "access-token",
            CONF_REPOS: [{"path": "home-assistant/core", "name": "HA Core"}],
        },
    )
    config_entry.add_to_hass(hass)
    assert await hass.config_entries.async_setup(config_entry.entry_id)
    await hass.async_block_till_done()

    # show initial form
    result = await hass.config_entries.options.async_init(config_entry.entry_id)
    # submit form with options
    result = await hass.config_entries.options.async_configure(
        result["flow_id"], user_input={"repos": []}
    )
    assert "create_entry" == result["type"]
    assert "" == result["title"]
    assert result["result"] is True
    assert {CONF_REPOS: []} == result["data"]
```

We first need to create a mock config entry and add it to Home Assistant. Next we generate
the initial options flow and capture the flow id. The flow id is used when we call
`hass.config_entries.options.async_configure` and pass in our `user_input` data. In this
case we are simulating unchecking the only repos that was configured.

Check out the [post on unit testing](https://aarongodfrey.dev/home%20automation/building_a_home_assistant_custom_component_part_2/) for more details on the fixtures and helpers used
here.

## Next Steps

At this point we now have a fully functional custom component that can be configured via
the configuration UI or a `configuration.yaml` file. In the last post in this series I
will briefly cover testing and debugging your component locally using the
[Visual Studio Code](https://code.visualstudio.com/) devcontainer provided by
[Home Assistant](https://developers.home-assistant.io/docs/development_environment#developing-with-visual-studio-code--devcontainer).
