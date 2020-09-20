---
title: "Bulding a Home Assistant Custom Component Part 2: Unit Testing and Continuous Integration"
excerpt: >
  Part 2 of building a custom component in Home Assistant.  In this post we'll
  examine how to setup unit testing and continous integration using GitHub actions.
  We'll also write some tests for our custom component.
categories:
  - home automation
tags:
  - homeassistant
  - development
  - tutorial
  - custom_component
  - testing
header:
  image: /assets/images/0013_header.png
  image_description: "GitHub Actions build status."
  caption: "GitHub Actions build status."
toc: true
---

<div>
  <p>This is the first part of a multi-part tutorial to create a Home Assistant custom component.</p>
  <ul>
    <li><a href="https://aarongodfrey.dev/home%20automation/building_a_home_assistant_custom_component_part_1/">Part 1 - Project Structure and Basics</a></li>
    <li>Part 2 - Unit Testing and Continuous Integration (Reading Now!)</li>
  </ul>
</div>
{: .notice--info}

## Introduction

In this post we will discuss how to unit test a custom component and
tie it all together using continuous integration in GitHub. We are still using
the same example project, [github-custom-component](https://github.com/boralyl/github-custom-component-tutorial).
I've added unit tests and some configuration for continuous integration. You can
find the diff for this post on the [feature/part2](https://github.com/boralyl/github-custom-component-tutorial/compare/feature/part1...feature/part2?expand=1)
branch.

## Unit Testing

Generally speaking there isn't anything special when it comes to testing a
Home Assistant custom component. The process is very similar to any other python
project. There are however several situations where having access to some
Home Assistant specific functionality makes writing tests much easier.

Home Assistant has a bunch of great test utilities and [pytest fixtures](https://docs.pytest.org/en/latest/fixture.html)
that make writing unit tests in the [core repo](https://github.com/home-assistant/core)
much simpler (like having access to a `hass` instance), but they are not exposed
anywhere that you can import them without copy/pasting the code. To make this
reusable for custom components, I created
a [pytest plugin](https://github.com/boralyl/pytest-homeassistant) that will
provide that functionality.

If you created your component using the
[cookiecutter project template](https://github.com/boralyl/cookiecutter-homeassistant-component)
for Home Assistant, than that requirement already exists in your `requirements.test.txt`.
If you have an existing component you can install it using pip.

```bash
$ pip install pytest-homeassistant
```

You don't need to do anything else to access the pytest fixtures that the `pytest-homeassistant`
plugin provides. pytest will automatically know about them and you can start
using them in your tests. One of the most useful is `hass` for providing a `hass`
instance that is properly setup for your test environment. This is especially
useful when testing your config flow. Check out the following example:

```python
async def test_flow_user_step_no_input(hass):
    """Test appropriate error when no input is provided."""
    _result = await hass.config_entries.flow.async_init(
        config_flow.DOMAIN, context={"source": "user"}
    )
    result = await hass.config_entries.flow.async_configure(
        _result["flow_id"], user_input={}
    )
    assert {"base": "missing"} == result["errors"]
```

When pytest sees an argument to your test function it will look it up based on
the name and all plugins registered. Since `pytest-homeassistant` registers this,
it will initialize it appropriately when the test function is called. We now
have the ability to run the different steps in the config flow with varying values
and make assertions about the data that is returned. In this particular case
we are testing that we display the appropriate error if the user did not provide
any input when configuring the component during the config flow process.

Another useful util from Home Assistant that `pytest-homeassistant` provides is
the `AsyncMock` for mocking return values of async functions. In this example
we are mocking the `github.getitem` async function to raise an exception.

```python
from pytest_homeassistant.async_mock import AsyncMock, MagicMock

from custom_components.github_custom.sensor import GitHubRepoSensor

async def test_async_update_failed():
    """Tests a failed async_update."""
    github = MagicMock()
    github.getitem = AsyncMock(side_effect=GitHubException)

    sensor = GitHubRepoSensor(github, {"path": "homeassistant/core"})
    await sensor.async_update()

    assert sensor.available is False
    assert {"path": "homeassistant/core"} == sensor.attrs
```

In this test we are verifying that if there is an exception raised in our
`async_update` function that we set the availability of our sensor to `False`.

I would reccomend reading through the unit tests for some of the
[platinum quiality](https://github.com/home-assistant/core/search?q=platinum&unscoped_q=platinum)
components in Home Assistant Core to get a better idea of what to test and how to
do it. As of writing this post those are:

- [Brother Printer](https://github.com/home-assistant/core/tree/dev/tests/components/brother)
- [Daikin AC](https://github.com/home-assistant/core/tree/dev/tests/components/daikin)
- [deCONZ](https://github.com/home-assistant/core/tree/dev/tests/components/deconz)
- [Elgato Key Light](https://github.com/home-assistant/core/tree/dev/tests/components/elgato)
- [Global Disaster Alert and Coordination System (GDACS](https://github.com/home-assistant/core/tree/dev/tests/components/gdacs)
- [GeoNet NZ Quakes](https://github.com/home-assistant/core/tree/dev/tests/components/geonetnz_quakes)
- [HomematicIP Cloud](https://github.com/home-assistant/core/tree/dev/tests/components/homematicip_cloud)
- [Philips Hue](https://github.com/home-assistant/core/tree/dev/tests/components/hue)
- [Internet Printing Protocol (IPP)](https://github.com/home-assistant/core/tree/dev/tests/components/ipp)
- [National Weather Service (NWS)](https://github.com/home-assistant/core/tree/dev/tests/components/nws)
- [Spain electricity hourly pricing (PVPC)](https://github.com/home-assistant/core/tree/dev/tests/components/pvpc_hourly_pricing)
- [Ubiquiti UniFi](https://github.com/home-assistant/core/tree/dev/tests/components/unifi)
- [VIZIO SmartCast](https://github.com/home-assistant/core/tree/dev/tests/components/vizio)
- [WLED](https://github.com/home-assistant/core/tree/dev/tests/components/wled)

You can also check out how I've implemented tests for three of my personal
custom components:

- [github-custom-component-tutorial](https://github.com/boralyl/github-custom-component-tutorial/tree/master/tests/)
- [nintendo-wishlist](https://github.com/custom-components/sensor.nintendo_wishlist/tree/master/tests)
- [steam-wishlist](https://github.com/boralyl/steam-wishlist/tree/master/tests)

## Continuous Integration

[Continous Integration](https://en.wikipedia.org/wiki/Continuous_integration) allows
running various checks on your code each time it is committed to the repository
(among other tasks). I'll specfically be talking about [GitHub Actions](https://github.com/features/actions)
in this post, but you can also use other services like [Travis CI](https://travis-ci.org/)
to achieve the same effect.

Your GitHub actions will live in the following folder structure in the root of
your repository: `.github/workflows/`. Each workflow is a different task to be
run, typically when pushing to GitHub.

### Hassfest

[@ludeeus](https://www.github.com/ludeeus) has created a GitHub action for validating
your component (also behind the fantastic [Home Assistant Community Store](https://hacs.xyz)).
Check out the blog post on the [Home Assistant Developers Blog](https://developers.home-assistant.io/blog/2020/04/16/hassfest/)
for more details.

Below are the contents of `.github/workflows/hassfest.yaml`

```yaml
name: Validate with hassfest

on:
  push:
  pull_request:
  schedule:
    - cron: "0 0 * * *"

jobs:
  validate:
    runs-on: "ubuntu-latest"
    steps:
      - uses: "actions/checkout@v2"
      - uses: home-assistant/actions/hassfest@master
```

This check essentially validates your custom component has a valid configuration.

### Python Build

For our python build, we will want to run all of our unit tests on every push.
Below are the contents of `.github/workflows/pythonpackage.yaml`:

```yaml
name: Python package

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.7]

    steps:
      - uses: actions/checkout@v1
      - name: Set up Python {% raw %}${{ matrix.python-version }}{% endraw %}
        uses: actions/setup-python@v1
        with:
          python-version: {% raw %}${{ matrix.python-version }}{% endraw %}

      - name: Set PY env
        run: echo "::set-env name=PY::$(python -VV | sha256sum | cut -d' ' -f1)"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.test.txt
      - name: Run pytest
        run: |
          pytest
```

What this does is that on each pushed commit it will set the default python
version for the environment to `3.7`. Then it will install our test requirements
and finally run `pytest` to execute our test suite. If there are no test failures,
the build will finish successfully and you will have a green check mark next to
your commit in GitHub (this assumes the `hassfest` check also succeeded).

## Pre-Commit

[pre-commit](https://pre-commit.com/) provides a way to check your code against
any number of checks prior to it being committed. This helps check for common
problems, format code to match the standard of the repository and many other checks.

If you used the cookie-cutter template than a [`.pre-commit-config.yaml`](https://github.com/boralyl/cookiecutter-homeassistant-component/blob/master/%7B%7B%20cookiecutter.domain%20%7D%7D/.pre-commit-config.yaml)
already exists in your generated code that contains most of the same checks that
the Home Assistant core repo uses. If you have an existing component, you can
easily add your own and pick and choose which checks you'd like to run against your
code.

The file generated by the cookie-cutter template assists in making your code more
compatible with Home Assistant's standards for when you want to merge your component
into the core repo.

To get started, add a `.pre-commit-config.yaml` to the root of your repository,
if one does not yet exist. Then install pre-commit.

```bash
$ pip install pre-commit
$ pre-commit install
```

Now next time you make a commit it will run all checks against the diff you are
attempting to commit and will fail if one of the checks fail. A successful
commit will look like the below output:

```bash
$ git commit -a
pyupgrade..........................Passed
black..............................Passed
codespell..........................Passed
flake8.............................Passed
bandit.............................Passed
isort..............................Passed
Check JSON.........................Passed
mypy...............................Passed
```

A failed check might look like:

```bash
$ git commit -a
pyupgrade........................(no files to check)Skipped
black............................(no files to check)Skipped
codespell........................Failed
- hook id: codespell
- exit code: 1

README.md:21: reccomend  ==> recommend

flake8...........................(no files to check)Skipped
bandit...........................(no files to check)Skipped
isort............................(no files to check)Skipped
Check JSON.......................(no files to check)Skipped
mypy.............................(no files to check)Skipped
```

Any check failures will abort your commit and you will need to fix any issues
and try to re-commit again. If for some reason you need to commit something and
want to bypass the checks you can specify the `--no-verify` flag when commiting.

```bash
$ git commit -a --no-verify
```

## Wrap Up

In this post we touched on how to start unit testing your custom component and
get it wired up with continuous integration using GithHub's workflows or other
third party solutions. Using these concepts will make your custom component not
only more robust and bug-free but also aligning better with the standards of
the Home Assistant core code.

In the next post we will look at how to add a [Config Flow](https://developers.home-assistant.io/docs/config_entries_config_flow_handler) to our [github-custom-component-tutorial](https://github.com/boralyl/github-custom-component-tutorial/) project.
