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

The subscribe terminal will show 25.

## Create a Virtual MQTT Device

Following javascript file create a device which generate a random number every 15 seconds.
```js
// mock-device.js
function getRandomFloat(min, max) {
    return Math.random() * (max - min) + min;
}

const deviceName = "MQ_DEVICE";
let message = "test-message";

// 1. Publish random number every 15 seconds
schedule('*/15 * * * * *', () => {
    let body = {
        "name": deviceName,
        "cmd": "randnum",
        "randnum": getRandomFloat(25,29).toFixed(1)
    };
    publish('DataTopic', JSON.stringify(body));
});

// 2. Receive the reading request, then return the response
// 3. Receive the put request, then change the device value
subscribe("CommandTopic" , (topic, val) => {
    var data = val;
    if (data.method == "set") {
        message = data[data.cmd]
    } else {
        switch(data.cmd) {
        case "ping":
            data.ping = "pong";
            break;
        case "message":
            data.message = message;
            break;
        case "randnum":
            data.randnum = 12.123;
            break;
        }
    }
    publish( "ResponseTopic", JSON.stringify(data));
});
```

Run with *mqtt-scripts*
```bash
$ docker run -d --restart always --name mqtt-scripts \
  -v /path/to/mqtt-scripts:/scripts dersimn/mqtt-scripts \
  --url mqtt://172.17.0.1 --dir /scripts
```

### Test Virtual Device

Now we should have docker container running as broker and device.
```bash
$ docker ps --format '{{.Image}}\t{{.Names}}'
eclipse-mosquitto       mqtt-broker
dersimn/mqtt-scripts    mqtt-scripts
```

To receive random number, we subscribe to topic *DataTopic*.
```bash
$ mosquitto_sub -t DataTopic
{"name":"MQ_DEVICE","cmd":"randnum","randnum":"27.6"}
{"name":"MQ_DEVICE","cmd":"randnum","randnum":"27.3"}
```

To verify *CommandTopic* and *ResponseTopic*, subscribe to *ResponseTopic* in one terminal,
```bash
$ mosquitto_sub -t ResponseTopic
```

Then publish to *CommandTopic* in another terminal,
```bash
$ mosquitto_pub -t CommandTopic -m test
```

## Start broker with docker compose and some issues of latest mosquitto

### Issues

1. Latest version of eclipse mosquitto broker won't accept anonymous users by default.
2. Mosquitto change ownership of empty volumns.

### Solutions
For issue #1, we need to customize mosquitto configure file.

```
allow_anonymous true # Anyone can connect

# Mosquitto v2+ requires that a listener be definer (otherwise it listens on loopback)
listener 1883


#log_type error
#log_type warning
#log_type notice
#log_type information
log_type all

#log_dest stdout
log_dest file /mosquitto/log/mosquitto.log

# Log entries are easier to read with an ISO 8601 timestamp
log_timestamp true
log_timestamp_format %Y-%m-%dT%H:%M:%S

# For demonstration purposes we will not store messages to disk (the appropriate value depends upon what you are testing)
# Note: If enabled then you will probably want to add a bind to the docker-compose.yml so the persistence_file is retained.
#persistence false

#persistence true
#autosave_interval 20
#persistence_location /mosquitto/data/
#persistence_file mosquitto.db

max_queued_messages 0
```

And create directory structure for mosquitto container. I added a readme.md file in each folder so that ownership of local folders wno't be changed to 1883 (issue #2).
```bash
tree ./mosquitto
./mosquitto/
├── config
│   └── mosquitto.conf
├── data
│   └── readme.md
└── log
    ├── mosquitto.log
    └── readme.md
```

Then customize our docker-compose.yml file.

```yml
version: "3"

networks:
  iot:

volumes:
  app_date: {}

services:
  mqtt:
    image: eclipse-mosquitto:2.0
    volumes:
      - type: bind
        source: ./mosquitto/config
        target: /mosquitto/config
        read_only: true
      - type: bind
        source: ./mosquitto/data
        target: /mosquitto/data
      - type: bind
        source: ./mosquitto/log
        target: /mosquitto/log
    ports:
      - 1883:1883
    networks:
      - iot
```

Now we can start eclipse mosquitto broker with docker compose.

```bash
$ docker-compose up -d
[+] Running 2/2
 ⠿ Network demo_iot       Created            0.0s
 ⠿ Container demo-mqtt-1  Started            0.4s
```
