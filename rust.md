# RUST LANGUAGE

## Install

## Tutorials

## Rust App as Minimal Docker Image
Create Dockerfile:

```Dockerfile
FROM rust:latest as build
WORKDIR /home/

RUN rustup target add x86_64-unknown-linux-musl

RUN USER=root cargo new helloworld
WORKDIR /home/helloworld
RUN cargo build --release

RUN cargo install --target x86_64-unknown-linux-musl --path .

FROM scratch as hello
COPY --from=build /usr/local/cargo/bin/helloworld .
USER 1000
CMD ["./helloworld"]
```

Then build docker image:

```bash
docker build -t hello .
```

The docker image _hello_ generated contains only statically linked app _helloworld_.
It's only 3.18MB compare to 1.16GB of rust image.
    
```sh
$ docker images
REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
hello                                          latest              4ba8b292a860        5 minutes ago       3.18MB
rust                                           latest              9b539306c373        17 hours ago        1.16GB
```

### [MUSL support for fully static binaries](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/musl-support-for-fully-static-binaries.html)
