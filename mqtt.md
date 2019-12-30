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
