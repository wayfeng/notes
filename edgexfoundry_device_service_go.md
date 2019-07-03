# EdgexFoundry: Build Device Service Go

### Download source code

```bash
$ git clone -b delhi https://github.com/edgexfoundry/device-sdk-go.git
$ cd device-sdk-go
```

### Use Aliyun mirror for alpine

Change *./example/cmd/device-simple/Dockerfile*.

```diff
--- a/example/cmd/device-simple/Dockerfile
+++ b/example/cmd/device-simple/Dockerfile
@@ -6,12 +6,13 @@
 FROM golang:1.11-alpine AS builder

 ENV GO111MODULE=on
-WORKDIR /go/src/github.com/edgexfoundry/device-sdk-go
+#WORKDIR /go/src/github.com/edgexfoundry/device-sdk-go
+WORKDIR /home/wayne/workspace/edgexfoundry/device-sdk-go

 LABEL license='SPDX-License-Identifier: Apache-2.0' \
   copyright='Copyright (c) 2018, 2019: Intel'

-RUN sed -e 's/dl-cdn[.]alpinelinux.org/nl.alpinelinux.org/g' -i~ /etc/apk/repositories
+RUN sed -e 's/dl-cdn[.]alpinelinux.org/mirrors.aliyun.com/g' -i~ /etc/apk/repositories

 # add git for go modules
 RUN apk update && apk add make git
```

### Make docker image

```bash
$ make docker
```