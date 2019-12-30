# MQTT

## Simple Demo

Start a MQTT broker.

```bash
$ docker run -d --rm --name mqtt-broker -p 1883:1883 eclipse-mosquitto
```

Install mosquitto client.
```bash
$ sudo apt install mosquitto-clients
```

Subscribe to the broker in one terminal.
```bash
$ mosquitto-sub -t sensors/temperature
```

Publish to the broker in another terminal.
```bash
$ mosquitto-pub -t sensors/temperature -m 25
```

The subscribe terminal will show =25=.
