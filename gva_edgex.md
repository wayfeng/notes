# GVA & EdgeX Demo

## Start MQTT Broker
```bash
$ docker run --name=broker -p 1883:1883 eclipse-mosquitto
```
### Verify MQTT Broker
Install mosquitto cli tools.
```bash
$ sudo apt install mosquitto-clients
```

In one terminal, subscribe to the broker, receive all topics.
```bash
$ mosquitto_sub -t '#'
```

In another terminal, publish a string to the broker. The string should appear in subscription terminal.
```bash
$ mosquitto_pub -t 'DataTopic' -m 'Hello world!'
```

## Build gst-video-analytics
First, check out GVA source code from github.com. We'll be using tag/v0.7.0 (with OpenVINO v2020.1) in this demo.
```bash
git clone https://github.com/opencv/gst-video-analytics.git
cd gst-video-analytics
git checkout tags/v0.7.0
```

Default ubuntu server may be slow, switch to local mirrors such as aliyun, by adding following lines to Dockerfile.
```Dockerfile
# Switch to Aliyun Mirror for apt
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
```

We'll need MQTT function in GVA, change Dockerfile to enable it.
```Dockerfile
#ARG ENABLE_PAHO_INSTALLATION=false
ARG ENABLE_PAHO_INSTALLATION=true
```

Build docker image.
```bash
cd docker
./build_docker_image.sh
```

After build complete, check GVA image. Image is quite big because many plugins and codecs are included. Image can be customized down in specific project.
```bash
$ docker images | grep gst
gst-video-analytics  latest  019ecb26659c  5 days ago  2.5GB
```

### Run GVA sample

Make sure all required models are downloaded.
```bash
$ git clone https://github.com/opencv/open_model_zoo.git
$ python3 ~/open_model_zoo/tools/downloader/downloader.py --list ~/gst-video-analytics/samples/model_downloader_configs/intel_models_for_samples.LST -o ~/edgex/data/models/intel
```

Start GVA container.
```
$ xhost local:root
$ docker run -it --privileged --network=host --name gva --device=/dev/video0 -v /tmp/.X11-unix/:/tmp/.X11-unix -e DISPLAY=$DISPLAY -v ~/edgex/data/models/intel:/root/intel_models:ro -e MODELS_PATH=/root/intel_models -v ~/edgex/data/video:/root/video-examples:ro -e VIDEO_EXAMPLES_DIR=/root/video-examples gst-video-analytics:latest
root@host:~# ls
gst-video-analytics  intel_models  video-examples
root@host:~# ./gst-video-analytics/samples/shell/face_detection_and_classification.sh /dev/video0
```

## Start EdgeX

Download EdgeXFoundry Fuji release.
```bash
$ mkdir edgex
$ cd edgex

$ curl https://raw.githubusercontent.com/edgexfoundry/developer-scripts/master/releases/fuji/compose-files/docker-compose-fuji-no-secty.yml -o ./docker-compose.yml
```

Add MQTT device service. [ref](https://fuji-docs.edgexfoundry.org/Ch-ExamplesAddingMQTTDevice.html)
```yml
  device-mqtt:
    image: edgexfoundry/docker-device-mqtt-go:1.1.1
    ports:
      - "49982:49982"
    container_name: edgex-device-mqtt
    hostname: edgex-device-mqtt
    networks:
      - edgex-network
    volumes:
      - db-data:/data/db
      - log-data:/edgex/logs
      - consul-config:/consul/config
      - consul-data:/consul/data
      - ./mqtt:/custom-config
    depends_on:
      - data
      - command
    entrypoint:
      - /device-mqtt
      - --registry=consul://edgex-core-consul:8500
      - --confdir=/custom-config
```

Create MQTT configurations.
```bash
$ mkdir mqtt
$ tree mqtt
mqtt/
├── configuration.toml
└── mqtt.test.device.profile.yml
```
MQTT configuration file.
```toml
# configuration.toml
[Writable]
LogLevel = 'DEBUG'

[Service]
Host = "edgex-device-mqtt"
Port = 49982
ConnectRetries = 3
Labels = []
OpenMsg = "device mqtt started"
Timeout = 5000
EnableAsyncReadings = true
AsyncBufferSize = 16

[Registry]
Host = "edgex-core-consul"
Port = 8500
CheckInterval = "10s"
FailLimit = 3
FailWaitTime = 10
Type = "consul"

[Logging]
EnableRemote = false
File = "./device-mqtt.log"

[Clients]
  [Clients.Data]
  Name = "edgex-core-data"
  Protocol = "http"
  Host = "edgex-core-data"
  Port = 48080
  Timeout = 50000

  [Clients.Metadata]
  Name = "edgex-core-metadata"
  Protocol = "http"
  Host = "edgex-core-metadata"
  Port = 48081
  Timeout = 50000

  [Clients.Logging]
  Name = "edgex-support-logging"
  Protocol = "http"
  Host ="edgex-support-logging"
  Port = 48061

[Device]
  DataTransform = true
  InitCmd = ""
  InitCmdArgs = ""
  MaxCmdOps = 128
  MaxCmdValueLen = 256
  RemoveCmd = ""
  RemoveCmdArgs = ""
  ProfilesDir = "/custom-config"

# Pre-define Devices
[[DeviceList]]
  Name = "MQ_GVA"
  Profile = "Device.MQTT.GVA"
  Description = "MQTT device for sending VA analytics data to EdgeX"
  Labels = ["MQTT"]
  [DeviceList.Protocols]
    [DeviceList.Protocols.mqtt]
       Schema = "tcp"
       Host = "172.17.0.1"
       Port = "1883"
       ClientId = "CommandPublisher"
       User = ""
       Password = ""
       Topic = "CommandTopic"

# Driver configs
[Driver]
IncomingSchema = "tcp"
IncomingHost = "172.17.0.1"
IncomingPort = "1883"
IncomingUser = ""
IncomingPassword = ""
IncomingQos = "0"
IncomingKeepAlive = "3600"
IncomingClientId = "IncomingDataSubscriber"
IncomingTopic = "DataTopic"
ResponseSchema = "tcp"
ResponseHost = "172.17.0.1"
ResponsePort = "1883"
ResponseUser = ""
ResponsePassword = ""
ResponseQos = "0"
ResponseKeepAlive = "3600"
ResponseClientId = "CommandResponseSubscriber"
ResponseTopic = "ResponseTopic"
```

MQTT device profile.
```yml
# mqtt.test.device.profile.yml
name: "Device.MQTT.GVA"
manufacturer: "Intel"
model: "MQTT-2"
labels:
- "test"
description: "Test device profile"

deviceResources:
- name: gvadata
  description: "inference result"
  attributes:
      { name: "gvadata" }
  properties:
    value:
        { type: "String", size: "0", readWrite: "R", defaultValue: "'{'test':'test'}'"  }
    units:
      { type: "String", readWrite: "R", defaultValue: "" }

deviceCommands:
- name: testgvadata
  get:
    - { index: "1", operation: "get", object: "gvadata", parameter: "gvadata", property: "value" }

coreCommands:
- name: testgvadata
  get:
    path: "/api/v1/device/{deviceId}/testgvadata"
    responses:
    -
      code: "200"
      description: "get video analytics data"
      expectedValues: ["gvadata"]
    -
      code: "503"
      description: "service unavailable"
      expectedValues: []
```

### EdgeX Feeder

