# esp8266-benq-rs232-mqtt
Objective: Control this [BenQ w1070 projector](https://www.benq.com/en-ap/support/downloads-faq/products/projector/w1070/manual.html) with an ESP8266 and MQTT via its RS232 interface and [Home Assistant](https://www.home-assistant.io/). I follow [BenQ's RS232 documentation](https://benqimage.blob.core.windows.net/driver-us-file/RS232-commands_all%20Product%20Lines.pdf).

## Adaption

Forked from [nicolaus-hee](https://github.com/nicolaus-hee/esp8266-benq-rs232-mqtt) who has a different projector and uses openHAB. I've made several small changes and use it with Home Assistant:

- Support setting lamp mode (eco, normal, etc)
- Support reading lamp hours
- Read the status of a connected reed switch

I've added a reed switch to the ESP, as the projector is in a cabinet and I'd like to base automations on if the cabinet is open or closed.

(TODO: image of the whole setup)

## What you need

* ESP8266 (I use model E12 on a nodeMCU)
* RS232 to TTL converter with female DB9 connector
* Basic wires

Connect pins like this:

ESP8266 | RS232-TTL
------- | ---------
G | GND
3V | VCC
D4 | TXD
RX | RXD

The reed switch is simply connected to 3V and D7, with a resistor (10kΩ) in paralell, like this:

<img src="https://github.com/RutgerKe/esp8266-benq-rs232-mqtt/blob/master/images/fritzing-reed.png" width="600" />

Source: [tutorial on a reed switch](https://randomnerdtutorials.com/door-status-monitor-using-the-esp8266/)

The end result, not too pretty but it works:

<img src="https://github.com/RutgerKe/esp8266-benq-rs232-mqtt/blob/master/images/hardware.jpg" width="600" />

## What the code does

* Read power / source / volume / lamp status from projector
* Send power (on/off), source change, lamp mode change and volume change commands to projector
* Publish status updates to MQTT server
* Listen for commands from MQTT server, then execute them
* Send custom commands via MQTT message
* Respond to custom commands via MQTT message

## How to use: MQTT

`stat` topics are published by the module, `cmnd` topics are listened to by the module and acted upon.

Topic | Payload | Comment
----- | ------- | --------
stat/projector/STATUS | {"POWER":"ON","SOURCE":"HDMI","VOLUME":"4","LAMP_HOUR":"3","LAMP_MODE":"ECO"} | Published every 5 seconds
cmnd/projector/POWER | ON, OFF | Power on or off
cmnd/projector/SOURCE | HDMI, SVID, VID, RGB, RGB2 | Set source / input
cmnd/projector/VOLUME | 0...20 | Set volume
cmnd/projector/LAMP | LNOR, ECO, SECO | Set lamp mode
cmnd/projector/COMMAND | --> | [Any command, e.g. vol=+](https://benqimage.blob.core.windows.net/driver-us-file/RS232-commands_all%20Product%20Lines.pdf)
stat/projector/COMMAND | {"COMMAND":"...","RESPONSE":"..."} | Returns result of above

## How to use: Home assistant

If you have not set up MQTT you can [read the home assistant docs](https://www.home-assistant.io/integrations/mqtt/) on how to do it. Add the below to your `configuration.yaml` and `automations.yaml`

```
# The on/off switch
switch:
  - platform: mqtt
    name: "Projector"
    state_topic: "stat/projector/STATUS"
    value_template: '{{ value_json.POWER }}'
    command_topic: "cmnd/projector/POWER"
    payload_on: "ON"
    payload_off: "OFF"
    icon: "mdi:projector"

# Define the valid lamp modes
input_select:
  projector_lamp_mode:
    name: Mode
    options:
      - "ECO"
      - "LNOR"
      - "SECO"
    icon: mdi:target

# The cabinet switch
binary_sensor:
  - platform: mqtt
    name: Projector cabinet
    state_topic: "stat/projector/STATUS"
    value_template: "{{ value_json.CABINET_DOOR }}"
    payload_on: "OPEN"
    payload_off: "CLOSED"

sensor:
    # Lamp hours sensor
  - platform: mqtt
    name: Lamp hours
    state_topic: "stat/projector/STATUS"
    unit_of_measurement: 'h'
    value_template: "{{ value_json.LAMP_HOURS }}"

    # A template switch to get the cabinet open/closed instead of on/off
    # Can probably be done in a nicer way, but this works.
  - platform: template
    sensors:
      projector_cabinet:
        value_template: '{% if states.binary_sensor.projector_cabinet %}
          {% if states.binary_sensor.projector_cabinet.state == "on" %}
            Open
          {% else %}
            Closed
          {% endif %}
          {% else %}
          n/a
          {% endif %}'
        friendly_name: 'Projector cabinet'

# Automations to update the input select when the projector changes,
# and change the projector when the input changes.
- id: projector_lamp_mode_get
  alias: Set projector lamp mode selector
  trigger:
    platform: mqtt
    topic: "stat/projector/STATUS"
  action:
    service: input_select.select_option
    data_template:
      entity_id: input_select.projector_lamp_mode
      # We don't want to trigger on "UNKNOWN" (not a valid option) or when the input is already at the same value (so we don't trigger the next automatino again, and end up in a loop. Yes that happend to me)
      option: "{{ trigger.payload_json.LAMP_MODE if trigger.payload_json.LAMP_MODE != 'UNKNOWN' and trigger.payload_json.LAMP_MODE !=  states(input_select.projector_lamp_mode) else states(input_select.projector_lamp_mode) }}"

# This automation script runs when the input mode is changed.
# It publishes the value to the command MQTT topic for the projector
- id: projector_lamp_mode_set
  alias: Set projector lamp mode
  trigger:
    platform: state
    entity_id: input_select.projector_lamp_mode
  action:
    service: mqtt.publish
    data_template:
      topic: "cmnd/projector/LAMP"
      retain: true
      payload: "{{ states('input_select.projector_lamp_mode') }}"%     
```

Note that not everything is configured, like the volume, but it would be simple to add. I simply don't need it as everything is connected to a networked receiver. The end result in home assistant can look like this:

<img src="https://github.com/RutgerKe/esp8266-benq-rs232-mqtt/blob/master/images/hass.png" width="344" />

## How to use: openHAB
 
This section is removed in the fork because it won't work with what I've added. You can see what was here in [a previous git version of this file](https://github.com/RutgerKe/esp8266-benq-rs232-mqtt/blob/1ee65c909a7f4023b69afa358b2b4877cadfdf14/README.md).