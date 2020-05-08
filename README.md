# MicroPython library to create sensors for HomeAssistant using MQTT Discovery

This is a simple MicroPython library which allows to create sensors and group of
sensors for HomeAssistant. The library makes use of the HomeAssistant [MQTT
Discovery](https://www.home-assistant.io/docs/mqtt/discovery/) method by
announcing the devices through a configuration topic.

## Getting Started

### Prerequisites

* MicroPython board capable of sending MQTT messages
* HomeAssistant with MQTT Broker running
* Some kind of sensor (not mandatory, but useful ;-))

### Usage

#### MQTT Client

First a MQTT Client needs to be created. The module
[umqtt.simple](https://github.com/micropython/micropython-lib/tree/master/umqtt.simple)
provides a simple MQTT client. The module can be installed using upip. Make
sure you are connected to the internet, e.g. using WiFi:

```
import network
wl = network.WLAN(network.STA_IF)
wl.active(True)
wl.connect(<SSID>, <PSK>)
wl.config(dhcp_hostname="Testboard")
```

Install the `umqtt.simple` module and create a MQTTClient object:
```
upip.install("micropython-umqtt.simple")
from umqtt.simple import MQTTClient
mqttc = MQTTClient(b"umqtt-testboard", <MQTT Server IP>, keepalive=10)
mqttc.connect()
```

#### Basic sensor

This example creates a single sensor and announces values. The second argument
to the `Sensor` constructor is the name argument and will be the entity name in
HomeAssistant. The argument `extra_config` allows to pass additional
configuration values as specified by the [MQTT Sensor
Component](https://www.home-assistant.io/integrations/sensor.mqtt/). In this
case the device class and unit of measurement of the sensor is specified.

With the creation of the Sensor object a persistent MQTT message is sent to the
discovery topic. The configuration will be picked up by HomeAssistant and a
sensor is created. The topic for the entites states update is handled by the
class.

The `publish_state()` function will send the MQTT message to the state topic
which will update the state of the particular entity in HomeAssistant.

```
# Temperature sensor...
import time
temp_config = { "unit_of_measurement": "°C", "device_class": "temperature" }
temp = Sensor(mqttc, b"temperature_sensor", b"sensorid", extra_conf=temp_config)
for i in range(10, 30):
    temp.publish_state(str(i))
    time.sleep(1)

# This deletes the sensor from HomeAssistant as well!
temp.remove_entity()
```

#### Multiple sensors

Multiple entities can share the same MQTT state topic. This allows to send a
single (JSON formatted) MQTT message to update multiple entites in
HomeAssistant. For this each entity needs to have a `value_template` specified
so each entity know which value it needs to parse.

```
group = EntityGroup(mqttc, b"testboard")
sensor1_config = { "unit_of_measurement": "°C", "device_class": "temperature",
    "value_template": "{{ value_json.temperature }}" }
sensor1 = group.create_sensor(b"test1", b"test1id", extra_conf=sensor1_config)

sensor2_config = { "unit_of_measurement": "%", "device_class": "humidity",
    "value_template": "{{ value_json.humidity }}" }
sensor2 = group.create_sensor(b"test2", b"test2id", extra_conf=sensor2_config)
```

To update the group the `publish_state()` function on the group object needs to
be used:
```
for i in range(10, 30):
    group.publish_state({ "temperature": str(i), "humidity": str(10 + i) })
    time.sleep(1)
```

To delete all entities of this group:
```
group.remove_group()
```

#### Multiple sensors with Device registry

HomeAssistant allows to group entites into devices via the [Device
Registry](https://developers.home-assistant.io/docs/device_registry_index/)
concept. MQTT entities allow to use the [device
registry](https://www.home-assistant.io/integrations/sensor.mqtt/#device) to
group individual entities to a device as well.

For this to work a `device` dictionary needs to be added to the configuration
variable. A `unique_id` property with a unique ID for each entity is required
too.

```
device_conf = { "identifiers": [ "common_identifier" ], "name": "Testboard",
    "manufacturer": "MicroPython", "model": "TinyPico", "sw_version": "0.1" }
common_conf = { "device": device_conf }
group = EntityGroup(mqttc, b"testboarder", extra_conf=common_conf)

sensor1_config = { "unit_of_measurement": "°C", "device_class":
    "temperature", "value_template": "{{ value_json.temperature }}"),
    "unique_id": mqtt_status + "sensor1"}
sensor1 = group.create_sensor(b"test1", b"test1id", extra_conf=sensor1_config)

sensor2_config = { "unit_of_measurement": "%", "device_class":
    "humidity", "value_template": "{{ value_json.humidity }}",
    "unique_id": mqtt_status + "sensor2"}
sensor2 = group.create_sensor(b"test2", b"test2id", extra_conf=sensor2_config)
```

Publishing state and removing the group stays the same as regular entity
groups.

#### Using availability topic

MQTT allows to specify a last will message which is sent to the specified topic
when a device does not connect for a certain period. The last will message is
transmitted when connecting, hence this needs to be configured before
connecting. Also choose a keep alive timeout which is higher than your expected
sensor update time. E.g. if you plan to update the sensor value every minute,
a value higher than 60 seconds make sense. The same topic then needs to be
specified when creating the sensor:

```
mqtt_availability_topic = b"testboard/status"
mqttc = MQTTClient(b"umqtt-testboard", <MQTT Server IP>, keepalive=70)
mqttc.set_last_will(mqtt_status, b"offline", True)
mqttc.connect()
mqttc.publish(mqtt_status, b"online", retain=True)

temp_config = { "availability_topic": mqtt_availability_topic }
temp = Sensor(mqttc, b"temperature_sensor", b"sensorid", extra_conf=temp_config)
```

## License

This project is licensed under the MIT License - see the
[LICENSE](LICENSE) file for details

