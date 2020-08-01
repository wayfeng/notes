# RUST LANGUAGE

## Install

## Tutorials

## Rust App as Minimal Docker Image
### With Ubuntu

Create Dockerfile:
```Dockerfile
FROM ubuntu:18.04 as builder

RUN apt-get update &&\
    apt-get install -y --no-install-recommends curl sudo
RUN useradd rust --user-group --create-home --shell /bin/bash --groups sudo
ADD sudoers /etc/sudoers.d/nopasswd

USER rust

ENV PATH=/home/rust/.cargo/bin:/usr/local/musl/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
RUN curl https://sh.rustup.rs -sSf | \
    sh -s -- -y
ADD cargo-config.toml /home/rust/.cargo/config

WORKDIR /home/rust/
RUN cargo new helloworld
WORKDIR /home/rust/helloworld
RUN cargo build --release && cargo install --path .

FROM ubuntu:18.04
COPY --from=builder /home/rust/helloworld/target/release/helloworld /usr/local/bin/
CMD ["helloworld"]
```

The image will be aroung 70MB.

### With Alpine Linux

```Dockerfile
FROM rust:1.45.0-alpine3.12 as builder
WORKDIR /home/

RUN USER=root cargo new helloworld
WORKDIR /home/helloworld
RUN cargo build --release

RUN cargo install --path .

FROM scratch as builder
COPY --from=build /usr/local/cargo/bin/helloworld .
USER 1000
CMD ["./helloworld"]
```

Then build docker image:

```bash
docker build -t hello .
```

The docker image _hello_ generated contains only statically linked app _helloworld_.
It's only 3.18MB.
    
```sh
$ docker images
REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
hello                                          latest              4ba8b292a860        5 minutes ago       3.18MB
rust                                           latest              9b539306c373        17 hours ago        1.16GB
```

### [MUSL support for fully static binaries](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/musl-support-for-fully-static-binaries.html)

The drawback of MUSL is not all C libraries can be built.
