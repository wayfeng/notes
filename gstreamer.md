---
marp: true
---

# Using OpenVINO with GStreamer
---
## OpenVINO Overview
---
## Model Optimizer
---
## Inference Engine
---
## Open Model Zoo
---
## Tools
---
# GStreamer Basics
---
## Pipeline
---
## Elements
- source or src
- sink
- mutex
- demutex
- tee
---
## Pads
- Always pads
- Sometimes pads
- Request pads
---
## Bins
- playbin
- decodebin
- etc.
---
## Tools
---
### gst-launch
```sh
# Play video file from remote server
$ gst-launch-1.0 playbin uri=http://g.io/a.mp4

# Play local file
$ gst-launch-1.0 playbin uri=file:///path/to/videos/b.mp4

# Custom pipeline
$ gst-launch-1.0 filesrc location=/path/to/videos/b.mp4 ! decodebin ! videoconvert ! autovideosink
```
---
### gst-inspect
```sh
# Show documents of an element
$ gst-inspect-1.0 filesrc

# find elements
$ gst-inspect-1.0 | grep 'gva'
```
---
### gst-typefind
```sh
$ gst-typefind-1.0 /path/to/videos/b.mp4
```
---
# GStreamer Plugins for OpenVINO
---
### Resources
https://github.com/opencv/gst-video-analytics
---
### Elements

[Details](https://github.com/opencv/gst-video-analytics/wiki/Elements)
- gvainference
- gvadetect
- gvaclassify
- gvatrack
- gvaidentify
- gvametaconvert
- gvametapublish
- gvawatermark
- gvafpscounter
- gvapython
---
#### Data Probe
[Details](https://gstreamer.freedesktop.org/documentation/application-development/advanced/pipeline-manipulation.html?gi-language=python#data-probes)