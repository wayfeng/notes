# Go Development


## Go with Docker

Docker can be a good environment for go developement. You have more versions to choose in [Docker Hub](hub.docker.com) without mess up your PC.

To get golang image with specific version.
```bash
$ docker pull golang:1.11-alpine
```

Run golang with interactive mode.
```bash
$ docker run -it golang:1.11-alpine
/go # ls
bin  src
```

Edit **go** file in local folder ./src/helloworld.go
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello world!");
}
```

You can send this file when start **go** docker.
```bash
$ docker run -it -v /path/to/src:/go/src golang:1.11-alpine
```

Or copy this file to a running container.
```bash
$ docker ps
CONTAINER ID        IMAGE
dcfd754d06f4        golang:1.11-alpine
$ docker cp /path/to/src/helloworld.go dcfd754d06f4:/go/src/
```
Then you can run this go file in the container.
```bash
/go # go run src/helloworld.go
Hello world!
```

## Go Proxy

Go website is blocked in PRC, to be able to use =go get= command to install dependencies, you will need Go proxy. To use Go proxy, =GO111MODULE= mode is also needed to be turned on.

```bash
if [ -d "/usr/local/go" ]; then
    PATH="$PATH:/usr/local/go/bin"
    export GO111MODULE=on
    export GOPROXY=https://goproxy.io
fi
```