---
layout: page
title: "Snips"
description: "Instructions on how to integrate Snips within Home Assistant."
date: 2018-05-02 12:00
sidebar: true
comments: false
sharing: true
footer: true
logo: snips.png
ha_category: Voice
ha_release: 0.48
---

The [Snips Voice Platform](https://www.snips.ai) allows users to add powerful voice assistants to their Raspberry Pi devices without compromising on privacy. It runs 100% on-device, and does not require an internet connection. It features Hotword Detection, Automatic Speech Recognition (ASR), Natural Language Understanding (NLU) and Dialog Management.

The latest documentation can be found here: [Snips Platform Documentation](https://snips.gitbook.io/documentation/).

![Snips Modules](/images/screenshots/snips_modules.png)

Snips takes voice or text as input and produces *intents* as output, which are explicit representations of an intention behind an utterance and which can subsequently be used by Home Assistant to perform appropriate actions.

![Snips Modules](/images/screenshots/snips_nlu.png)


## {% linkable_title The Snips Voice Platform %}

### {% linkable_title Installation %}

The Snips platform can be installed via the Snips APT/Debian repository. If you prefer to install the platform using the Docker distribution, check out our [Docker Installation Guide](https://github.com/snipsco/snips-platform-documentation/wiki/6.--Miscellaneous#using-docker).

```bash
$ sudo apt-get update
$ sudo apt-get install -y dirmngr
$ sudo bash -c  'echo "deb https://raspbian.snips.ai/$(lsb_release -cs) stable main" > /etc/apt/sources.list.d/snips.list'
$ sudo apt-key adv --keyserver pgp.mit.edu --recv-keys D4F50CDCA10A2849
$ sudo apt-get update
$ sudo apt-get install -y snips-platform-voice
```

Note: if the keyserver pgp.mit.edu is down try to use another one in the 4th line , like pgp.surfnet.nl:

```bash
sudo apt-key adv --keyserver pgp.surfnet.nl --recv-keys D4F50CDCA10A2849
```

### {% linkable_title Creating an assistant %}

Head over to the [Snips Console](https://console.snips.ai) to create your assistant. Launch the training and download by clicking on the "Download Assistant" button.

The next step is to get the assistant to work on your device. Unzip and copy the assistant folder that you downloaded from the web console to the path. Assuming your downloaded assistant folder is on your desktop, just run:

```bash
$ scp -r ~/Desktop/assistant pi@<raspi_hostname.local_or_IP>:/home/pi/.
```

Now ssh into your Raspberry Pi:

```bash
$ ssh pi@<raspi_hostname.local_or_IP>
```

By default, this command is `ssh pi@raspberrypi.local`, if you are using the default Raspberry Pi hostname.

Then, move the assistant to the right folder:

```bash
(pi) $ sudo mv /home/pi/assistant /usr/share/snips/assistant
```

Note: if you already have an assistant installed and wish to replace it, start by removing the previous one, and then move the new one in its place:

```bash
(pi) $ sudo rm -r /usr/share/snips/assistant
(pi) $ sudo mv /home/pi/assistant /usr/share/snips/assistant
```

### {% linkable_title Running Snips %}

Make sure that a microphone is plugged to the Raspberry Pi. If you are having trouble setting up audio, we have written a guide on [Raspberry Pi Audio Configuration](https://snips.gitbook.io/documentation/installing-snips/on-a-raspberry-pi#2-configuration).

Start the Snips Voice Platform by starting the `snips-*` services:

```bash
$ sudo systemctl start "snips-*"
```

Snips is now ready to take voice commands from the microphone. To trigger the listening, simply say

> Hey Snips

followed by a command, e.g.

> Set the lights to green in the living room

As the Snips Platform parses this query into an intent, it will be published on MQTT, on the `hermes/intent/<intentName>` topic. The Snips Home Assistant component subscribes to this topic, and handles the intent according to the rules defined in `configuration.yaml` file, as explained below.

#### {% linkable_title Optional: specifying an external MQTT broker %}

By default, Snips runs its own MQTT broker. But we can also tell Snips to use an external broker by specifying this when launching Snips. In this case, we need to specify this in the `/etc/snips.toml` configuration file. For more information on configuring this, see the [Using an external MQTT broker](https://snips.gitbook.io/documentation/advanced-configuration/platform-configuration) article.

## {% linkable_title Home Assistant configuration %}

{% configuration %}
feedback_sounds:
  description: Turn on feedbacks sounds for Snips.
  required: false
  type: str
  default: false
site_ids:
  description: A list of siteIds if using multiple Snips instances. Used to make sure feedback is toggled on or off for all sites.
  required: false
  type: str
probability_threshold:
  description: Threshold for intent probability. Range is from 0.00 to 1.00, 1 being highest match. Intents under this level are discarded.
  require: false
  type: float
{% endconfiguration %}

### {% linkable_title Specifying the MQTT broker %}

Messages between Snips and Home Assistant are passed via MQTT. We can either point Snips to the MQTT broker used by Home Assistant, as explained above, or tell Home Assistant which [MQTT broker](/docs/mqtt/) to use by adding the following entry to the `configuration.yaml` file:

```yaml
mqtt:
  broker: MQTT_BROKER_IP
  port: MQTT_BROKER_PORT
```

By default, Snips runs an MQTT broker on port 9898. So if we wish to use this broker, and if Snips and Home Assistant run on the same device, the entry will look as follows:

```yaml
mqtt:
  broker: 127.0.0.1
  port: 9898
```

Alternatively, MQTT can be configured to bridge messages between servers if using a custom MQTT broker such as [mosquitto](https://mosquitto.org/).

### {% linkable_title Triggering actions %}

In Home Assistant, we trigger actions based on intents produced by Snips using the [`intent_script`](/components/intent_script) component. For instance, the following block handles a `ActivateLightColor` intent to change light colors:

Note: If your Snips action is prefixed with a username (e.g. `john:playmusic` or `john__playmusic`), the Snips component in Home Assistant [will try and strip off the username](https://github.com/home-assistant/home-assistant/blob/c664c20165ebeb248b98716cf61e865f274a2dac/homeassistant/components/snips.py#L126-L129). Bear this in mind if you get the error `Received unknown intent` even when what you see on the MQTT bus looks correct. Internally the Snips component is trying to match the non-username version of the intent (i.e., just `playmusic`).

{% raw %}
```yaml
snips:

intent_script:
  ActivateLightColor:
    action:
      - service: light.turn_on
        data_template:
          entity_id: light.{{ objectLocation | replace(" ","_") }}
          color_name: {{ objectColor }}
```
{% endraw %}

In the `data_template` block, we have access to special variables, corresponding to the slot names for the intent. In the present case, the `ActivateLightColor` has two slots, `objectLocation` and `objectColor`.

### {% linkable_title Special slots %}

Two special values for slots are populated with the siteId the intent originated from and the probability value for the intent.

In the above example, the slots are plain strings. However, snips has a duration builtin value used for setting timers and this will be parsed to a seconds value.

{% raw %}
```yaml
SetTimer:
  speech:
    type: plain
    text: weather
  action:
    service: script.set_timer
    data_template:
      name: "{{ timer_name }}"
      duration: "{{ timer_duration }}"
      siteId: "{{ site_id }}"
      probability: "{{ probability }}"
```
{% endraw %}



### {% linkable_title Sending TTS Notifications %}

You can send TTS notifications to Snips using the snips.say and snips.say_action services. Say_action starts a session and waits for user response, "Would you like me to close the garage door?", "Yes, close the garage door".

#### {% linkable_title Service `snips.say` %}

| Service data attribute | Optional | Description                                            |
|------------------------|----------|--------------------------------------------------------|
| `text`                 |       no | Text to say.                                           |
| `site_id`              |      yes | Site to use to start session.                          |
| `custom_data`          |      yes | custom data that will be included with all messages in this session. |

#### {% linkable_title Service `snips.say_action` %}

| Service data attribute | Optional | Description                                            |
|------------------------|----------|--------------------------------------------------------|
| `text`                 |       no | Text to say.                                           |
| `site_id`              |      yes | Site to use to start session.                          |
| `custom_data`          |      yes | custom data that will be included with all messages in this session. |
| `can_be_enqueued`      |      yes | If True, session waits for an open session to end, if False session is dropped if one is running. |
| `intent_filter`        |      yes | Array of Strings - A list of intents names to restrict the NLU resolution to on the first query. |


### {% linkable_title Snips Support %}

There is an active [discord](https://discordapp.com/invite/3939Kqx) channel for further support.

### {% linkable_title Configuration Examples %}

#### {% linkable_title Turn on a light %}

```yaml
intent_script:
  turn_on_light:
    speech:
      type: plain
      text: 'OK, turning on the light'
    action:
      service: light.turn_on
```

##### {% linkable_title Open a Garage Door %}

```yaml
intent_script:
  OpenGarageDoor:
    speech:
      type: plain
      text: 'OK, opening the garage door'
    action:
      - service: cover.open_cover
        data:
          entity_id: garage_door
```

##### {% linkable_title Intiating a query %}

Here is a more complex example. The automation is triggered if the garage door is open for more than 10 minutes.
Snips will then ask you if you want to close it and if you respond with something like "Close the garage door" it
will do so. Unfortunately there is no builtin support for yes and no responses.

```yaml
automation:
  garage_door_has_been_open:
    trigger:
     - platform: state
        entity_id: binary_sensor.my_garage_door_sensor
        from: 'off'
        to: 'on'
        for:
          minutes: 10
    sequence:
      service: snips.say_action
        data:
          text: 'Garage door has been open 10 minutes, would you like me to close it?'
          intentFilter:
            - closeGarageDoor

# This intent is fired if the user responds with the appropriate intent after the above notification
intent_script:
  closeGarageDoor:
    speech:
      type: plain
      text: 'OK, closing the garage door'
    action:
      - service: script.garage_door_close
```

##### {% linkable_title Weather %}

So now you can open and close your garage door, let's check the weather. Add the Weather by Snips Skill to your assistant. Create a weather sensor, in this example [Dark Sk](/components/sensor.darksky/) and the `api_key` in the `secrets.yaml` file.

```yaml
- platform: darksky
  name: "Dark Sky Weather"
  api_key: !secret dark_sky_key
  update_interval:
    minutes: 10
  monitored_conditions:
    - summary
    - hourly_summary
    - temperature
    - temperature_max
    - temperature_min
```

Then add this to your configuration file.

{% raw %}
```yaml
intent_script:
  searchWeatherForecast:
    speech:
      type: plain
      text: >
        The weather is currently
        {{ states('sensor.dark_sky_weather_temperature') | round(0) }}
        degrees outside and {{ states('sensor.dark_sky_weather_summary') }}.
        The high today will be
        {{ states('sensor.dark_sky_weather_daily_high_temperature') | round(0)}}
        and {{ states('sensor.dark_sky_weather_hourly_summary') }}
```
{% endraw %}

